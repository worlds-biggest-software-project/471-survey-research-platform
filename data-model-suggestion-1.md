# Data Model Suggestion 1: Normalized Relational Database (PostgreSQL)

> Project: Survey & Research Platform (Candidate #471)
> Generated: 2026-05-26

## Overview

This model uses a fully normalized relational schema in PostgreSQL, following Third Normal Form (3NF) principles. Every entity — organizations, users, surveys, questions, responses, panels, experiments, and analytics — is represented as a distinct table with explicit foreign-key relationships and referential integrity constraints. This approach prioritises data consistency, query flexibility, and compliance auditability, which are critical requirements for a research-grade survey platform handling PII and statistical data.

## Technology Stack

| Component | Technology | Rationale |
|-----------|-----------|-----------|
| Primary Database | PostgreSQL 17+ | Mature RDBMS with advanced indexing, partitioning, full-text search, and JSONB support as escape hatch |
| ORM | Prisma 6.x or TypeORM | Type-safe query building, migration management, schema-as-code |
| Migration Tool | Prisma Migrate or Flyway | Versioned, repeatable migrations with rollback support |
| Connection Pooling | PgBouncer or Supabase Pooler | Transaction-mode pooling for high-concurrency web workloads |
| Search | PostgreSQL tsvector + GIN indexes | Full-text search on survey titles, question text, open-text responses |
| Caching | Redis 7+ | Session cache, rate limiting, real-time dashboard aggregation cache |

---

## Complete Schema Definition

### 1. Multi-Tenancy & Identity

```sql
-- ============================================================
-- ORGANIZATIONS & TEAMS
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
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    deleted_at          TIMESTAMPTZ
);

CREATE INDEX idx_organizations_slug ON organizations (slug);

CREATE TABLE users (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email               VARCHAR(255) NOT NULL UNIQUE,
    email_verified      BOOLEAN NOT NULL DEFAULT FALSE,
    password_hash       VARCHAR(255),
    full_name           VARCHAR(255) NOT NULL,
    avatar_url          TEXT,
    locale              VARCHAR(10) NOT NULL DEFAULT 'en',
    timezone            VARCHAR(50) NOT NULL DEFAULT 'UTC',
    last_login_at       TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    deleted_at          TIMESTAMPTZ
);

CREATE INDEX idx_users_email ON users (email);

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
-- TEAM WORKSPACES
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
    role                VARCHAR(30) NOT NULL DEFAULT 'member'
                        CHECK (role IN ('admin', 'editor', 'analyst', 'member', 'viewer')),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (workspace_id, user_id)
);
```

### 2. Survey Design & Structure

```sql
-- ============================================================
-- SURVEYS
-- ============================================================

CREATE TABLE surveys (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id     UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    workspace_id        UUID REFERENCES workspaces(id) ON DELETE SET NULL,
    title               VARCHAR(500) NOT NULL,
    description         TEXT,
    internal_name       VARCHAR(255),
    status              VARCHAR(30) NOT NULL DEFAULT 'draft'
                        CHECK (status IN ('draft', 'scheduled', 'active', 'paused', 'closed', 'archived')),
    survey_type         VARCHAR(50) NOT NULL DEFAULT 'standard'
                        CHECK (survey_type IN ('standard', 'nps', 'csat', 'ces', 'conjoint',
                                               'maxdiff', 'experiment', 'longitudinal')),
    default_language    VARCHAR(10) NOT NULL DEFAULT 'en',
    allow_back          BOOLEAN NOT NULL DEFAULT TRUE,
    show_progress_bar   BOOLEAN NOT NULL DEFAULT TRUE,
    randomize_questions BOOLEAN NOT NULL DEFAULT FALSE,
    anonymous_responses BOOLEAN NOT NULL DEFAULT FALSE,
    response_limit      INTEGER,
    starts_at           TIMESTAMPTZ,
    closes_at           TIMESTAMPTZ,
    estimated_duration  INTEGER,  -- seconds
    completion_rate_prediction DECIMAL(5,2),
    created_by          UUID NOT NULL REFERENCES users(id),
    published_at        TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    deleted_at          TIMESTAMPTZ
);

CREATE INDEX idx_surveys_org ON surveys (organization_id);
CREATE INDEX idx_surveys_status ON surveys (status);
CREATE INDEX idx_surveys_type ON surveys (survey_type);
CREATE INDEX idx_surveys_workspace ON surveys (workspace_id);

-- ============================================================
-- SURVEY TRANSLATIONS
-- ============================================================

CREATE TABLE survey_translations (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    survey_id           UUID NOT NULL REFERENCES surveys(id) ON DELETE CASCADE,
    language            VARCHAR(10) NOT NULL,
    title               VARCHAR(500) NOT NULL,
    description         TEXT,
    thank_you_message   TEXT,
    is_rtl              BOOLEAN NOT NULL DEFAULT FALSE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (survey_id, language)
);

-- ============================================================
-- PAGES & SECTIONS
-- ============================================================

CREATE TABLE survey_pages (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    survey_id           UUID NOT NULL REFERENCES surveys(id) ON DELETE CASCADE,
    title               VARCHAR(255),
    description         TEXT,
    sort_order          INTEGER NOT NULL DEFAULT 0,
    randomize           BOOLEAN NOT NULL DEFAULT FALSE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_survey_pages_survey ON survey_pages (survey_id, sort_order);

-- ============================================================
-- QUESTIONS
-- ============================================================

CREATE TABLE questions (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    survey_id           UUID NOT NULL REFERENCES surveys(id) ON DELETE CASCADE,
    page_id             UUID NOT NULL REFERENCES survey_pages(id) ON DELETE CASCADE,
    question_type       VARCHAR(50) NOT NULL
                        CHECK (question_type IN (
                            'single_choice', 'multiple_choice', 'dropdown',
                            'likert', 'likert_matrix', 'nps',
                            'text_short', 'text_long', 'text_email', 'text_number',
                            'ranking', 'rating', 'slider',
                            'date', 'date_time',
                            'file_upload',
                            'matrix_single', 'matrix_multiple',
                            'conjoint_profile', 'maxdiff_set',
                            'semantic_differential',
                            'constant_sum',
                            'image_choice',
                            'net_promoter_score'
                        )),
    question_code       VARCHAR(50),  -- e.g. Q1, Q2a — researcher-assigned code
    title               TEXT NOT NULL,
    description         TEXT,
    is_required         BOOLEAN NOT NULL DEFAULT FALSE,
    sort_order          INTEGER NOT NULL DEFAULT 0,
    randomize_options   BOOLEAN NOT NULL DEFAULT FALSE,
    allow_other         BOOLEAN NOT NULL DEFAULT FALSE,
    other_label         VARCHAR(255),
    min_selections      INTEGER,
    max_selections      INTEGER,
    min_value           DECIMAL(15,4),
    max_value           DECIMAL(15,4),
    step_value          DECIMAL(15,4),
    validation_regex    VARCHAR(500),
    validation_message  VARCHAR(500),
    placeholder_text    VARCHAR(255),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_questions_survey ON questions (survey_id);
CREATE INDEX idx_questions_page ON questions (page_id, sort_order);
CREATE INDEX idx_questions_type ON questions (question_type);

-- ============================================================
-- QUESTION TRANSLATIONS
-- ============================================================

CREATE TABLE question_translations (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    question_id         UUID NOT NULL REFERENCES questions(id) ON DELETE CASCADE,
    language            VARCHAR(10) NOT NULL,
    title               TEXT NOT NULL,
    description         TEXT,
    placeholder_text    VARCHAR(255),
    validation_message  VARCHAR(500),
    other_label         VARCHAR(255),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (question_id, language)
);

-- ============================================================
-- ANSWER OPTIONS (for choice-based questions)
-- ============================================================

CREATE TABLE answer_options (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    question_id         UUID NOT NULL REFERENCES questions(id) ON DELETE CASCADE,
    label               TEXT NOT NULL,
    value               VARCHAR(255) NOT NULL,
    sort_order          INTEGER NOT NULL DEFAULT 0,
    is_exclusive        BOOLEAN NOT NULL DEFAULT FALSE,  -- "None of the above"
    is_anchored         BOOLEAN NOT NULL DEFAULT FALSE,  -- excluded from randomization
    score               DECIMAL(10,4),                   -- for scoring/weighting
    image_url           TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_answer_options_question ON answer_options (question_id, sort_order);

CREATE TABLE answer_option_translations (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    answer_option_id    UUID NOT NULL REFERENCES answer_options(id) ON DELETE CASCADE,
    language            VARCHAR(10) NOT NULL,
    label               TEXT NOT NULL,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (answer_option_id, language)
);

-- ============================================================
-- MATRIX ROWS (for matrix / Likert-matrix questions)
-- ============================================================

CREATE TABLE matrix_rows (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    question_id         UUID NOT NULL REFERENCES questions(id) ON DELETE CASCADE,
    label               TEXT NOT NULL,
    sort_order          INTEGER NOT NULL DEFAULT 0,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_matrix_rows_question ON matrix_rows (question_id, sort_order);

CREATE TABLE matrix_row_translations (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    matrix_row_id       UUID NOT NULL REFERENCES matrix_rows(id) ON DELETE CASCADE,
    language            VARCHAR(10) NOT NULL,
    label               TEXT NOT NULL,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (matrix_row_id, language)
);

-- ============================================================
-- BRANCHING LOGIC & SKIP PATTERNS
-- ============================================================

CREATE TABLE logic_rules (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    survey_id           UUID NOT NULL REFERENCES surveys(id) ON DELETE CASCADE,
    source_question_id  UUID NOT NULL REFERENCES questions(id) ON DELETE CASCADE,
    rule_type           VARCHAR(30) NOT NULL
                        CHECK (rule_type IN ('skip_to', 'show_question', 'hide_question',
                                             'show_page', 'hide_page', 'end_survey',
                                             'set_variable', 'redirect')),
    target_question_id  UUID REFERENCES questions(id) ON DELETE CASCADE,
    target_page_id      UUID REFERENCES survey_pages(id) ON DELETE CASCADE,
    condition_operator  VARCHAR(30) NOT NULL
                        CHECK (condition_operator IN ('equals', 'not_equals', 'contains',
                                                       'not_contains', 'greater_than',
                                                       'less_than', 'is_answered',
                                                       'is_not_answered', 'in_set')),
    condition_value     TEXT,
    condition_values    TEXT[],  -- for 'in_set' operator
    logic_group         INTEGER NOT NULL DEFAULT 0,  -- for AND/OR grouping
    logic_connector     VARCHAR(5) NOT NULL DEFAULT 'AND'
                        CHECK (logic_connector IN ('AND', 'OR')),
    sort_order          INTEGER NOT NULL DEFAULT 0,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_logic_rules_survey ON logic_rules (survey_id);
CREATE INDEX idx_logic_rules_source ON logic_rules (source_question_id);
```

### 3. Distribution & Collection

```sql
-- ============================================================
-- DISTRIBUTION CHANNELS
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
    settings_json       TEXT,  -- channel-specific config (kept as TEXT for strict normalization)
    response_count      INTEGER NOT NULL DEFAULT 0,
    created_by          UUID NOT NULL REFERENCES users(id),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_dist_channels_survey ON distribution_channels (survey_id);
CREATE INDEX idx_dist_channels_slug ON distribution_channels (unique_link_slug);

-- ============================================================
-- EMAIL INVITATIONS
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
    last_reminded_at    TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_email_invitations_channel ON email_invitations (channel_id);
CREATE INDEX idx_email_invitations_token ON email_invitations (token);
CREATE INDEX idx_email_invitations_status ON email_invitations (status);

-- ============================================================
-- IN-APP TRIGGERS
-- ============================================================

CREATE TABLE in_app_triggers (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    channel_id          UUID NOT NULL REFERENCES distribution_channels(id) ON DELETE CASCADE,
    event_name          VARCHAR(255) NOT NULL,
    event_property      VARCHAR(255),
    event_operator      VARCHAR(30),
    event_value         VARCHAR(255),
    user_segment        VARCHAR(255),
    delay_seconds       INTEGER NOT NULL DEFAULT 0,
    max_displays        INTEGER NOT NULL DEFAULT 1,
    cooldown_hours      INTEGER NOT NULL DEFAULT 24,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### 4. Responses & Answer Storage

```sql
-- ============================================================
-- SURVEY RESPONSES (the core response record)
-- ============================================================

CREATE TABLE survey_responses (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    survey_id           UUID NOT NULL REFERENCES surveys(id) ON DELETE CASCADE,
    channel_id          UUID REFERENCES distribution_channels(id) ON DELETE SET NULL,
    respondent_id       UUID REFERENCES panel_respondents(id) ON DELETE SET NULL,
    external_user_id    VARCHAR(255),  -- for in-app surveys
    session_id          VARCHAR(100),
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
    weight              DECIMAL(10,6) DEFAULT 1.0,  -- sample weight
    started_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    completed_at        TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);

-- Partition by quarter for high-volume response tables
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
CREATE INDEX idx_responses_completed ON survey_responses (completed_at)
    WHERE completed_at IS NOT NULL;

-- ============================================================
-- INDIVIDUAL ANSWERS (one row per question per response)
-- ============================================================

CREATE TABLE response_answers (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    response_id         UUID NOT NULL REFERENCES survey_responses(id) ON DELETE CASCADE,
    question_id         UUID NOT NULL REFERENCES questions(id) ON DELETE CASCADE,
    answer_option_id    UUID REFERENCES answer_options(id),      -- for choice questions
    matrix_row_id       UUID REFERENCES matrix_rows(id),          -- for matrix questions
    text_value          TEXT,                                       -- open-text answers
    numeric_value       DECIMAL(15,4),                             -- numeric answers, slider, NPS
    date_value          TIMESTAMPTZ,                               -- date answers
    array_value         TEXT[],                                    -- multiple selections
    file_url            TEXT,                                       -- file upload answers
    duration_ms         INTEGER,                                   -- time spent on this question
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);

CREATE TABLE response_answers_2026_q1 PARTITION OF response_answers
    FOR VALUES FROM ('2026-01-01') TO ('2026-04-01');
CREATE TABLE response_answers_2026_q2 PARTITION OF response_answers
    FOR VALUES FROM ('2026-04-01') TO ('2026-07-01');
CREATE TABLE response_answers_2026_q3 PARTITION OF response_answers
    FOR VALUES FROM ('2026-07-01') TO ('2026-10-01');
CREATE TABLE response_answers_2026_q4 PARTITION OF response_answers
    FOR VALUES FROM ('2026-10-01') TO ('2027-01-01');

CREATE INDEX idx_response_answers_response ON response_answers (response_id);
CREATE INDEX idx_response_answers_question ON response_answers (question_id);
CREATE INDEX idx_response_answers_option ON response_answers (answer_option_id)
    WHERE answer_option_id IS NOT NULL;
CREATE INDEX idx_response_answers_numeric ON response_answers (question_id, numeric_value)
    WHERE numeric_value IS NOT NULL;

-- ============================================================
-- RANKING ANSWERS (preserving rank order)
-- ============================================================

CREATE TABLE response_rankings (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    response_id         UUID NOT NULL REFERENCES survey_responses(id) ON DELETE CASCADE,
    question_id         UUID NOT NULL REFERENCES questions(id) ON DELETE CASCADE,
    answer_option_id    UUID NOT NULL REFERENCES answer_options(id),
    rank_position       INTEGER NOT NULL,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (response_id, question_id, rank_position)
);

-- ============================================================
-- CONSTANT SUM ANSWERS
-- ============================================================

CREATE TABLE response_constant_sums (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    response_id         UUID NOT NULL REFERENCES survey_responses(id) ON DELETE CASCADE,
    question_id         UUID NOT NULL REFERENCES questions(id) ON DELETE CASCADE,
    answer_option_id    UUID NOT NULL REFERENCES answer_options(id),
    allocated_value     DECIMAL(15,4) NOT NULL,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (response_id, question_id, answer_option_id)
);
```

### 5. Panel Management

```sql
-- ============================================================
-- PANELS & RESPONDENTS
-- ============================================================

CREATE TABLE panels (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id     UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    name                VARCHAR(255) NOT NULL,
    description         TEXT,
    panel_type          VARCHAR(30) NOT NULL DEFAULT 'internal'
                        CHECK (panel_type IN ('internal', 'external', 'mixed')),
    source              VARCHAR(50),
    is_active           BOOLEAN NOT NULL DEFAULT TRUE,
    member_count        INTEGER NOT NULL DEFAULT 0,
    created_by          UUID NOT NULL REFERENCES users(id),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_panels_org ON panels (organization_id);

CREATE TABLE panel_respondents (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    panel_id            UUID NOT NULL REFERENCES panels(id) ON DELETE CASCADE,
    email               VARCHAR(255),
    phone               VARCHAR(50),
    external_id         VARCHAR(255),
    first_name          VARCHAR(255),
    last_name           VARCHAR(255),
    gender              VARCHAR(20),
    birth_year          INTEGER,
    country             VARCHAR(3),
    region              VARCHAR(100),
    city                VARCHAR(100),
    language            VARCHAR(10),
    occupation          VARCHAR(255),
    income_bracket      VARCHAR(50),
    education_level     VARCHAR(50),
    consent_given       BOOLEAN NOT NULL DEFAULT FALSE,
    consent_given_at    TIMESTAMPTZ,
    opt_out             BOOLEAN NOT NULL DEFAULT FALSE,
    opt_out_at          TIMESTAMPTZ,
    total_surveys_sent  INTEGER NOT NULL DEFAULT 0,
    total_surveys_completed INTEGER NOT NULL DEFAULT 0,
    last_survey_at      TIMESTAMPTZ,
    quality_score       DECIMAL(5,2),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_panel_respondents_panel ON panel_respondents (panel_id);
CREATE INDEX idx_panel_respondents_email ON panel_respondents (email);
CREATE INDEX idx_panel_respondents_country ON panel_respondents (country);
CREATE INDEX idx_panel_respondents_quality ON panel_respondents (quality_score DESC)
    WHERE quality_score IS NOT NULL;

-- ============================================================
-- PANEL CUSTOM ATTRIBUTES (normalized EAV for extensible demographics)
-- ============================================================

CREATE TABLE panel_attribute_definitions (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    panel_id            UUID NOT NULL REFERENCES panels(id) ON DELETE CASCADE,
    attribute_name      VARCHAR(100) NOT NULL,
    attribute_type      VARCHAR(30) NOT NULL
                        CHECK (attribute_type IN ('text', 'number', 'date', 'boolean',
                                                   'single_choice', 'multiple_choice')),
    allowed_values      TEXT[],
    is_required         BOOLEAN NOT NULL DEFAULT FALSE,
    sort_order          INTEGER NOT NULL DEFAULT 0,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (panel_id, attribute_name)
);

CREATE TABLE panel_respondent_attributes (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    respondent_id       UUID NOT NULL REFERENCES panel_respondents(id) ON DELETE CASCADE,
    attribute_id        UUID NOT NULL REFERENCES panel_attribute_definitions(id) ON DELETE CASCADE,
    text_value          TEXT,
    numeric_value       DECIMAL(15,4),
    date_value          DATE,
    boolean_value       BOOLEAN,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (respondent_id, attribute_id)
);

CREATE INDEX idx_respondent_attrs_respondent ON panel_respondent_attributes (respondent_id);
CREATE INDEX idx_respondent_attrs_attribute ON panel_respondent_attributes (attribute_id);

-- ============================================================
-- QUOTAS
-- ============================================================

CREATE TABLE survey_quotas (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    survey_id           UUID NOT NULL REFERENCES surveys(id) ON DELETE CASCADE,
    name                VARCHAR(255) NOT NULL,
    target_count        INTEGER NOT NULL,
    current_count       INTEGER NOT NULL DEFAULT 0,
    is_active           BOOLEAN NOT NULL DEFAULT TRUE,
    action_on_full      VARCHAR(30) NOT NULL DEFAULT 'screen_out'
                        CHECK (action_on_full IN ('screen_out', 'end_survey', 'redirect')),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE quota_conditions (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    quota_id            UUID NOT NULL REFERENCES survey_quotas(id) ON DELETE CASCADE,
    question_id         UUID NOT NULL REFERENCES questions(id) ON DELETE CASCADE,
    operator            VARCHAR(30) NOT NULL,
    value               TEXT NOT NULL,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- LONGITUDINAL WAVES
-- ============================================================

CREATE TABLE longitudinal_studies (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id     UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    name                VARCHAR(255) NOT NULL,
    description         TEXT,
    panel_id            UUID REFERENCES panels(id),
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
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (study_id, wave_number)
);

CREATE INDEX idx_study_waves_study ON study_waves (study_id, wave_number);
```

### 6. Advanced Research Methods

```sql
-- ============================================================
-- CONJOINT ANALYSIS
-- ============================================================

CREATE TABLE conjoint_designs (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    survey_id           UUID NOT NULL REFERENCES surveys(id) ON DELETE CASCADE,
    design_type         VARCHAR(30) NOT NULL DEFAULT 'choice_based'
                        CHECK (design_type IN ('choice_based', 'adaptive', 'full_profile')),
    num_tasks           INTEGER NOT NULL DEFAULT 10,
    profiles_per_task   INTEGER NOT NULL DEFAULT 3,
    include_none        BOOLEAN NOT NULL DEFAULT FALSE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE conjoint_attributes (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    design_id           UUID NOT NULL REFERENCES conjoint_designs(id) ON DELETE CASCADE,
    attribute_name      VARCHAR(255) NOT NULL,
    sort_order          INTEGER NOT NULL DEFAULT 0,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE conjoint_levels (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    attribute_id        UUID NOT NULL REFERENCES conjoint_attributes(id) ON DELETE CASCADE,
    level_label         VARCHAR(255) NOT NULL,
    sort_order          INTEGER NOT NULL DEFAULT 0,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE conjoint_tasks (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    design_id           UUID NOT NULL REFERENCES conjoint_designs(id) ON DELETE CASCADE,
    task_number         INTEGER NOT NULL,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE conjoint_profiles (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    task_id             UUID NOT NULL REFERENCES conjoint_tasks(id) ON DELETE CASCADE,
    profile_number      INTEGER NOT NULL,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE conjoint_profile_levels (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    profile_id          UUID NOT NULL REFERENCES conjoint_profiles(id) ON DELETE CASCADE,
    attribute_id        UUID NOT NULL REFERENCES conjoint_attributes(id) ON DELETE CASCADE,
    level_id            UUID NOT NULL REFERENCES conjoint_levels(id) ON DELETE CASCADE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (profile_id, attribute_id)
);

CREATE TABLE conjoint_responses (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    response_id         UUID NOT NULL REFERENCES survey_responses(id) ON DELETE CASCADE,
    task_id             UUID NOT NULL REFERENCES conjoint_tasks(id) ON DELETE CASCADE,
    chosen_profile_id   UUID REFERENCES conjoint_profiles(id),  -- NULL if "none" chosen
    response_time_ms    INTEGER,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (response_id, task_id)
);

-- ============================================================
-- MAXDIFF ANALYSIS
-- ============================================================

CREATE TABLE maxdiff_designs (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    survey_id           UUID NOT NULL REFERENCES surveys(id) ON DELETE CASCADE,
    num_sets            INTEGER NOT NULL DEFAULT 10,
    items_per_set       INTEGER NOT NULL DEFAULT 5,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE maxdiff_items (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    design_id           UUID NOT NULL REFERENCES maxdiff_designs(id) ON DELETE CASCADE,
    item_label          VARCHAR(500) NOT NULL,
    sort_order          INTEGER NOT NULL DEFAULT 0,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE maxdiff_sets (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    design_id           UUID NOT NULL REFERENCES maxdiff_designs(id) ON DELETE CASCADE,
    set_number          INTEGER NOT NULL,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE maxdiff_set_items (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    set_id              UUID NOT NULL REFERENCES maxdiff_sets(id) ON DELETE CASCADE,
    item_id             UUID NOT NULL REFERENCES maxdiff_items(id) ON DELETE CASCADE,
    position            INTEGER NOT NULL,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (set_id, item_id)
);

CREATE TABLE maxdiff_responses (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    response_id         UUID NOT NULL REFERENCES survey_responses(id) ON DELETE CASCADE,
    set_id              UUID NOT NULL REFERENCES maxdiff_sets(id) ON DELETE CASCADE,
    best_item_id        UUID NOT NULL REFERENCES maxdiff_items(id),
    worst_item_id       UUID NOT NULL REFERENCES maxdiff_items(id),
    response_time_ms    INTEGER,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (response_id, set_id)
);
```

### 7. Experiment Design

```sql
-- ============================================================
-- A/B EXPERIMENTS
-- ============================================================

CREATE TABLE experiments (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    survey_id           UUID NOT NULL REFERENCES surveys(id) ON DELETE CASCADE,
    name                VARCHAR(255) NOT NULL,
    description         TEXT,
    hypothesis          TEXT,
    status              VARCHAR(30) NOT NULL DEFAULT 'draft'
                        CHECK (status IN ('draft', 'running', 'completed', 'cancelled')),
    allocation_method   VARCHAR(30) NOT NULL DEFAULT 'random'
                        CHECK (allocation_method IN ('random', 'stratified', 'blocked')),
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

### 8. Analysis & AI Features

```sql
-- ============================================================
-- OPEN-TEXT ANALYSIS
-- ============================================================

CREATE TABLE text_analysis_jobs (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    survey_id           UUID NOT NULL REFERENCES surveys(id) ON DELETE CASCADE,
    question_id         UUID NOT NULL REFERENCES questions(id) ON DELETE CASCADE,
    status              VARCHAR(30) NOT NULL DEFAULT 'pending'
                        CHECK (status IN ('pending', 'processing', 'completed', 'failed')),
    model_version       VARCHAR(100),
    response_count      INTEGER NOT NULL DEFAULT 0,
    started_at          TIMESTAMPTZ,
    completed_at        TIMESTAMPTZ,
    created_by          UUID NOT NULL REFERENCES users(id),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE text_themes (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    job_id              UUID NOT NULL REFERENCES text_analysis_jobs(id) ON DELETE CASCADE,
    theme_label         VARCHAR(500) NOT NULL,
    theme_description   TEXT,
    sentiment           VARCHAR(20)
                        CHECK (sentiment IN ('positive', 'negative', 'neutral', 'mixed')),
    response_count      INTEGER NOT NULL DEFAULT 0,
    percentage          DECIMAL(5,2),
    confidence_score    DECIMAL(5,4),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE text_theme_responses (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    theme_id            UUID NOT NULL REFERENCES text_themes(id) ON DELETE CASCADE,
    response_answer_id  UUID NOT NULL REFERENCES response_answers(id) ON DELETE CASCADE,
    relevance_score     DECIMAL(5,4),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (theme_id, response_answer_id)
);

-- ============================================================
-- STATISTICAL ANALYSIS RESULTS
-- ============================================================

CREATE TABLE analysis_runs (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    survey_id           UUID NOT NULL REFERENCES surveys(id) ON DELETE CASCADE,
    analysis_type       VARCHAR(50) NOT NULL
                        CHECK (analysis_type IN ('cross_tabulation', 'significance_test',
                                                  'regression', 'anova', 'conjoint_utility',
                                                  'maxdiff_scoring', 'nps_trend',
                                                  'sample_weighting', 'sentiment_analysis')),
    parameters          TEXT NOT NULL,  -- serialized analysis parameters
    status              VARCHAR(30) NOT NULL DEFAULT 'pending'
                        CHECK (status IN ('pending', 'running', 'completed', 'failed')),
    result_summary      TEXT,
    created_by          UUID NOT NULL REFERENCES users(id),
    started_at          TIMESTAMPTZ,
    completed_at        TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE analysis_results (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    run_id              UUID NOT NULL REFERENCES analysis_runs(id) ON DELETE CASCADE,
    result_key          VARCHAR(255) NOT NULL,
    result_value        TEXT NOT NULL,
    numeric_result      DECIMAL(20,10),
    p_value             DECIMAL(10,8),
    confidence_interval_low  DECIMAL(15,6),
    confidence_interval_high DECIMAL(15,6),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_analysis_results_run ON analysis_results (run_id);

-- ============================================================
-- AI-GENERATED REPORTS
-- ============================================================

CREATE TABLE generated_reports (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    survey_id           UUID NOT NULL REFERENCES surveys(id) ON DELETE CASCADE,
    report_type         VARCHAR(30) NOT NULL
                        CHECK (report_type IN ('executive_summary', 'detailed_analysis',
                                                'trend_report', 'comparison_report')),
    title               VARCHAR(500) NOT NULL,
    content_markdown    TEXT NOT NULL,
    model_used          VARCHAR(100),
    status              VARCHAR(30) NOT NULL DEFAULT 'draft',
    created_by          UUID NOT NULL REFERENCES users(id),
    published_at        TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### 9. Integrations & Webhooks

```sql
-- ============================================================
-- INTEGRATIONS
-- ============================================================

CREATE TABLE integrations (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id     UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    integration_type    VARCHAR(50) NOT NULL
                        CHECK (integration_type IN ('salesforce', 'hubspot', 'slack',
                                                     'tableau', 'zapier', 'webhook',
                                                     'google_sheets', 'snowflake',
                                                     'bigquery', 'custom_api')),
    name                VARCHAR(255) NOT NULL,
    is_active           BOOLEAN NOT NULL DEFAULT TRUE,
    config_encrypted    BYTEA,  -- encrypted configuration blob
    last_sync_at        TIMESTAMPTZ,
    last_error          TEXT,
    created_by          UUID NOT NULL REFERENCES users(id),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE webhooks (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id     UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    survey_id           UUID REFERENCES surveys(id) ON DELETE CASCADE,
    url                 TEXT NOT NULL,
    secret              VARCHAR(255) NOT NULL,
    events              TEXT[] NOT NULL,  -- e.g. {'response.completed', 'survey.closed'}
    is_active           BOOLEAN NOT NULL DEFAULT TRUE,
    last_triggered_at   TIMESTAMPTZ,
    failure_count       INTEGER NOT NULL DEFAULT 0,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- AUDIT LOG
-- ============================================================

CREATE TABLE audit_log (
    id                  BIGSERIAL PRIMARY KEY,
    organization_id     UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    user_id             UUID REFERENCES users(id),
    action              VARCHAR(100) NOT NULL,
    entity_type         VARCHAR(50) NOT NULL,
    entity_id           UUID,
    old_values          TEXT,
    new_values          TEXT,
    ip_address          INET,
    user_agent          TEXT,
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

CREATE INDEX idx_audit_log_org ON audit_log (organization_id, created_at);
CREATE INDEX idx_audit_log_entity ON audit_log (entity_type, entity_id);
CREATE INDEX idx_audit_log_user ON audit_log (user_id);
```

### 10. GDPR & Compliance

```sql
-- ============================================================
-- CONSENT RECORDS
-- ============================================================

CREATE TABLE consent_records (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    respondent_id       UUID REFERENCES panel_respondents(id) ON DELETE SET NULL,
    response_id         UUID REFERENCES survey_responses(id) ON DELETE SET NULL,
    consent_type        VARCHAR(50) NOT NULL
                        CHECK (consent_type IN ('data_collection', 'data_processing',
                                                 'marketing', 'research_participation',
                                                 'data_transfer', 'cookie_tracking')),
    consent_given       BOOLEAN NOT NULL,
    consent_text        TEXT NOT NULL,
    ip_address          INET,
    given_at            TIMESTAMPTZ NOT NULL DEFAULT now(),
    withdrawn_at        TIMESTAMPTZ,
    legal_basis         VARCHAR(50) NOT NULL DEFAULT 'consent'
                        CHECK (legal_basis IN ('consent', 'legitimate_interest',
                                                'contractual', 'legal_obligation'))
);

CREATE INDEX idx_consent_respondent ON consent_records (respondent_id);
CREATE INDEX idx_consent_response ON consent_records (response_id);

-- ============================================================
-- DATA DELETION REQUESTS (GDPR Article 17)
-- ============================================================

CREATE TABLE data_deletion_requests (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id     UUID NOT NULL REFERENCES organizations(id),
    requester_email     VARCHAR(255) NOT NULL,
    respondent_id       UUID REFERENCES panel_respondents(id),
    request_type        VARCHAR(30) NOT NULL
                        CHECK (request_type IN ('erasure', 'export', 'rectification')),
    status              VARCHAR(30) NOT NULL DEFAULT 'pending'
                        CHECK (status IN ('pending', 'in_progress', 'completed', 'denied')),
    reason              TEXT,
    processed_by        UUID REFERENCES users(id),
    processed_at        TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Entity Relationship Summary

```
organizations 1──* workspaces 1──* surveys
organizations 1──* users (via organization_memberships)
organizations 1──* panels 1──* panel_respondents

surveys 1──* survey_pages 1──* questions 1──* answer_options
surveys 1──* distribution_channels 1──* email_invitations
surveys 1──* survey_responses 1──* response_answers
surveys 1──* survey_quotas 1──* quota_conditions
surveys 1──* conjoint_designs / maxdiff_designs
surveys 1──* experiments 1──* experiment_groups

questions *──1 answer_options (via response_answers)
questions 1──* matrix_rows

survey_responses *──1 panel_respondents
survey_responses 1──* response_answers
survey_responses 1──* response_rankings
survey_responses 1──* conjoint_responses / maxdiff_responses

longitudinal_studies 1──* study_waves *──1 surveys
```

---

## Pros and Cons

### Pros

1. **Referential integrity everywhere.** Foreign keys enforce data consistency across the entire research lifecycle — from survey design through panel recruitment, response collection, and analysis. A survey cannot be deleted while active responses reference it; a question cannot reference a nonexistent page.

2. **Mature tooling ecosystem.** PostgreSQL is the most widely supported open-source RDBMS. Prisma, TypeORM, Sequelize, Drizzle, and Knex all provide excellent PostgreSQL support. Migration tooling, monitoring (pganalyze, Datadog), and managed hosting (Supabase, Neon, AWS RDS, GCP Cloud SQL) are production-proven.

3. **Strong compliance story.** The explicit schema makes it straightforward to implement GDPR erasure (cascade deletes with audit), data export (structured joins), and audit trails. Compliance auditors can inspect the schema and verify data flows.

4. **Optimised for analytical queries.** Cross-tabulation, statistical aggregation, and respondent filtering are natural SQL operations. PostgreSQL window functions, CTEs, and aggregate functions handle significance testing, weighted averages, and cohort analysis natively.

5. **Time-tested partitioning.** PostgreSQL declarative partitioning on `created_at` for high-volume tables (responses, answers, audit log) provides predictable performance scaling and enables efficient data retention policies.

6. **Type safety end-to-end.** Prisma or TypeORM generate TypeScript types directly from the schema, catching data-model errors at compile time rather than runtime.

### Cons

1. **Schema rigidity for evolving question types.** Adding a new question type (e.g., heatmap, card sort) requires adding columns to the `questions` table and potentially new answer tables. Each new format is a migration.

2. **Wide answer table.** `response_answers` uses nullable columns (`text_value`, `numeric_value`, `date_value`, `array_value`, `file_url`) to accommodate different question types. Most rows use only one or two of these columns, wasting some storage.

3. **Conjoint/MaxDiff table proliferation.** Advanced research methods require many related tables (designs, attributes, levels, tasks, profiles, profile-levels, responses). This makes the schema complex for research-intensive deployments.

4. **Migration burden at scale.** Schema changes on tables with hundreds of millions of rows (e.g., adding a column to `response_answers`) require careful zero-downtime migration strategies (e.g., `pg_repack`, lazy backfills).

5. **Horizontal scaling limits.** PostgreSQL does not natively shard across nodes. At very high volume (billions of responses), you need Citus, read replicas, or an analytical data warehouse sidecar.

6. **Multilingual overhead.** Separate translation tables (`survey_translations`, `question_translations`, `answer_option_translations`, `matrix_row_translations`) add join complexity for every localized query.

---

## Migration and Scaling Considerations

### Phase 1: Single-Instance PostgreSQL (MVP to 10M responses)
- Single PostgreSQL 17 instance on managed hosting (Supabase, Neon, or AWS RDS)
- PgBouncer for connection pooling (transaction mode)
- Quarterly partitioning on `survey_responses`, `response_answers`, `audit_log`
- Redis for session cache and real-time dashboard counters
- Estimated cost: $50-200/month for database hosting

### Phase 2: Read Replicas + Analytical Sidecar (10M to 500M responses)
- Primary PostgreSQL for writes; 1-2 streaming replicas for read-heavy dashboard and reporting queries
- Consider pg_cron for automated partition creation
- Offload heavy analytical queries (cross-tabulation across millions of rows) to a columnar store (ClickHouse, DuckDB) via logical replication
- Implement materialized views for common dashboard aggregations (NPS trends, completion rates by channel)
- Estimated cost: $500-2,000/month

### Phase 3: Distributed PostgreSQL (500M+ responses)
- Citus extension for horizontal sharding by `organization_id`
- Multi-region deployment with data residency enforcement (EU data stays on EU nodes)
- Archive old partitions to S3-backed columnar storage (Parquet format) for long-term retention
- Dedicated analytical cluster for conjoint utility estimation and regression analysis
- Estimated cost: $5,000-20,000/month

### Zero-Downtime Migration Strategy
1. Use `ALTER TABLE ... ADD COLUMN` with defaults (non-blocking in PostgreSQL 11+)
2. Backfill new columns lazily with batch UPDATE statements during low-traffic windows
3. Create new tables and migrate data in parallel, then swap with `ALTER TABLE ... RENAME`
4. Use Prisma Migrate or Flyway for versioned, reviewable migration scripts
5. All migrations must be tested against a production-size data snapshot before deployment
