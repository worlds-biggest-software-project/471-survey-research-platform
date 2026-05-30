# Survey & Research Platform — Phased Development Plan

> Project: 471-survey-research-platform · Created: 2026-05-30
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

This plan synthesises `research.md`, `features.md`, `standards.md`, `README.md`, and the four `data-model-suggestion-*.md` documents into a concrete, phased build. The core value proposition — a mid-market survey platform that unifies design, distribution, statistical analysis, and AI text analysis at a price below Qualtrics — ships across Phases 3–6. Advanced research methods (conjoint/MaxDiff) and enterprise integrations come later.

---

## Core Requirements (synthesised)

- **What it does**: Full-lifecycle survey research — design (drag-and-drop builder, branching logic, AI question assistant), distribute (web link, email, QR, in-app), collect (anonymous or panel-linked responses with quotas), analyse (real-time dashboard, cross-tabs, significance tests, conjoint/MaxDiff, AI open-text theme analysis), and report (live dashboards, branded PDF, AI executive summaries).
- **Who uses it**: Product managers, market researchers, HR/people-ops, and CX programme owners on mid-market teams priced out of Qualtrics but underserved by SurveyMonkey/Typeform's analytical depth.
- **Key differentiators**: (1) Advanced statistics (conjoint, MaxDiff, significance testing) at mid-market price; (2) AI text analysis without a data-science team; (3) per-response pricing; (4) self-hostable for data sovereignty; (5) MCP server + OpenAPI for AI-agent integration.
- **Deployment model**: Hybrid — SaaS cloud (multi-tenant) and self-hosted (Docker Compose / Helm), single codebase. Data residency selectable per organisation.
- **Standards to implement**: OpenAPI 3.1 (public API), JSON Schema Draft 2020-12 (survey definition validation), OAuth 2.0 + JWT (RFC 6749/7519) and Bearer tokens (RFC 6750) for the API, WCAG 2.2 AA for the respondent runtime, GDPR/CCPA tooling (consent ledger, erasure, export), OWASP ASVS L2 controls, SCIM 2.0 (RFC 7643/7644) for enterprise provisioning (backlog), MCP server for agents.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Language (backend) | TypeScript on Node.js 22 LTS | The domain is API/realtime/integration-heavy, not ML-training-heavy. A single language across the API, the survey-runtime renderer, the builder UI, and the MCP server minimises context-switching and lets survey-definition types be shared end-to-end. Matches the ecosystem of the closest open-source comparator (Formbricks). |
| Language (statistics) | Python 3.12 microservice (`scipy`, `statsmodels`, `numpy`, `pandas`, `pylogit`/`xlogit`) | Significance testing, regression, ANOVA, conjoint utility estimation, and MaxDiff scoring are first-class in the Python scientific stack and have no patent barriers (per `features.md` legal summary). Isolating them in a service keeps heavy numeric deps out of the Node runtime and lets the analysis engine scale independently. |
| API framework | NestJS 11 (Fastify adapter) | Opinionated module structure scales to a large multi-domain codebase; first-class dependency injection, guards (RBAC), interceptors (audit), and `@nestjs/swagger` auto-generates the **OpenAPI 3.1** document required by `standards.md`. |
| Survey runtime renderer | SurveyJS `survey-core` + `survey-react-ui` (MIT) | MIT-licensed, JSON-schema-driven rendering engine that already implements the table-stakes question types, branching, and WCAG-conformant form patterns — avoids reimplementing the respondent runtime. Our survey-definition JSONB maps onto its model. |
| Builder + dashboard UI | Next.js 15 (App Router) + React 19 + Tailwind + shadcn/ui | Server components for fast dashboard loads; SurveyJS Creator embeds for the drag-and-drop builder; one deployable web app. |
| Primary database | PostgreSQL 17 — **Hybrid Relational + JSONB** (data-model-suggestion-3) | Chosen over fully-normalised (suggestion 1: rigid for evolving question types), event-sourced (suggestion 2: too much MVP complexity), and polyglot (suggestion 4: heavy ops). Stable entities (orgs, users, panels, responses) are relational for integrity and aggregation; variable entities (survey definitions, question configs, answer payloads, analysis results) are JSONB validated by Zod. This mirrors how Formbricks and SurveyJS actually model surveys. |
| ORM / migrations | Prisma 6 | Type-safe access to relational + `Json` fields; versioned, reviewable migrations with rollback. |
| Validation | Zod | Runtime validation of JSONB survey-definition and answer payloads against versioned schemas; shared types between API and UI. |
| Task queue | BullMQ on Redis 7 | Async workloads: email/SMS sending, webhook delivery, AI calls (text analysis, summaries), statistical analysis jobs, scheduled digests, abandonment detection. Durable, retryable, observable. |
| Cache / realtime | Redis 7 | Dashboard counters, quota tracking, rate limiting, session state, pub/sub for live dashboard updates over WebSocket. |
| Analytics store (scale phase) | DuckDB embedded for MVP; ClickHouse pluggable later (suggestion 4) | Cross-tab and statistical aggregation run in PostgreSQL/DuckDB for MVP; the polyglot CDC path is the documented Phase-3 scaling escape hatch, not MVP scope. |
| Object storage | S3-compatible (MinIO for self-host) | File-upload answers, generated PDF/PPT reports, response archive exports. |
| LLM provider | Anthropic Claude via a provider-abstraction layer (`AiProvider` interface) | AI question assistant, completion-rate prediction, open-text theme analysis, executive summaries. Abstraction allows self-hosters to swap in a local/OpenAI-compatible model. |
| Email / SMS | Pluggable: SMTP / SES for email, Twilio for SMS, behind `MessagingProvider` interface | Self-host friendly; cloud uses managed providers. |
| Auth | Auth.js (NextAuth) for UI sessions; OAuth 2.0 + JWT + API keys for the public API | Implements RFC 6749/7519/6750 from `standards.md`. SCIM 2.0 is a backlog enterprise add-on. |
| AI-agent surface | MCP server (`@modelcontextprotocol/sdk`) wrapping the public API | Differentiation opportunity flagged in `standards.md`: expose create-survey, fetch-responses, run-analysis tools to research agents. |
| Containerisation | Docker + docker-compose (dev/self-host); Helm chart (k8s) | First-class self-hosted deployment per README. |
| Testing | Vitest (unit/integration, TS), Playwright (E2E + WCAG axe-core scans), pytest (Python analysis service), Testcontainers (real Postgres/Redis) | Standard per ecosystem; axe-core enforces WCAG 2.2 AA. |
| Code quality | ESLint + Prettier + `tsc --noEmit` (TS); ruff + mypy (Python) | Enforced in CI and Definition of Done. |
| Package manager | pnpm workspaces (monorepo) + uv (Python service) | Monorepo holds API, web, shared types, MCP server; Python service is a sibling package. |

### Project Structure

```
survey-research-platform/
├── package.json                      # pnpm workspace root
├── pnpm-workspace.yaml
├── docker-compose.yml                # postgres, redis, minio, api, web, analysis, mailpit
├── docker-compose.prod.yml
├── Dockerfile.api
├── Dockerfile.web
├── Dockerfile.analysis
├── deploy/
│   └── helm/                         # Kubernetes Helm chart
├── packages/
│   ├── shared/                       # @srp/shared — Zod schemas + TS types shared across apps
│   │   └── src/
│   │       ├── survey-definition.ts  # SurveyDefinition Zod schema (the JSONB contract)
│   │       ├── answer-payloads.ts    # Per-question-type answer Zod schemas
│   │       ├── events.ts             # Domain event names + payloads
│   │       └── api-types.ts          # DTOs shared between api and web
│   ├── api/                          # @srp/api — NestJS application
│   │   ├── prisma/
│   │   │   ├── schema.prisma
│   │   │   └── migrations/
│   │   └── src/
│   │       ├── main.ts
│   │       ├── app.module.ts
│   │       ├── common/               # guards, interceptors (audit, tenancy), pipes, filters
│   │       ├── auth/                 # Auth.js bridge, OAuth2, JWT, API keys, RBAC
│   │       ├── orgs/                 # organizations, memberships, workspaces, billing tier
│   │       ├── surveys/              # survey CRUD, definition validation, versioning, publish
│   │       ├── builder-ai/           # AI question assistant, completion-rate prediction
│   │       ├── distribution/         # channels, email/sms invitations, QR, in-app triggers
│   │       ├── runtime/              # respondent-facing collection API (public, unauth)
│   │       ├── responses/            # response + answer storage, export (CSV/SPSS)
│   │       ├── panels/               # panels, respondents, attributes, quotas, longitudinal
│   │       ├── analysis/             # bridge to Python analysis service, job orchestration
│   │       ├── text-ai/             # open-text theme analysis, executive summaries
│   │       ├── experiments/          # A/B + control group management
│   │       ├── dashboard/            # realtime aggregation, websocket gateway
│   │       ├── reports/              # PDF/PPT generation, scheduled digests, share links
│   │       ├── integrations/         # webhooks, Salesforce/HubSpot/Slack connectors, Zapier
│   │       ├── compliance/           # consent ledger, GDPR erasure/export, audit query
│   │       ├── queue/                # BullMQ processors
│   │       └── mcp/                  # MCP server tools (mounted alongside API)
│   ├── web/                          # @srp/web — Next.js builder + dashboard + respondent runtime
│   │   └── src/app/
│   │       ├── (app)/                # authenticated builder + dashboard
│   │       └── s/[slug]/             # public respondent runtime (SurveyJS render)
│   └── mcp/                          # @srp/mcp — standalone MCP server binary (optional)
├── services/
│   └── analysis/                     # Python FastAPI statistical engine
│       ├── pyproject.toml
│       ├── app/
│       │   ├── main.py
│       │   ├── crosstab.py
│       │   ├── significance.py
│       │   ├── regression.py
│       │   ├── conjoint.py
│       │   ├── maxdiff.py
│       │   └── weighting.py
│       └── tests/
└── tests/
    └── e2e/                          # Playwright end-to-end + accessibility scans
```

---

## Phase 1: Foundation, Tenancy & Auth

### Purpose
Establish the monorepo, the PostgreSQL hybrid schema for stable entities, multi-tenant organisation/user/workspace model, authentication (sessions + API keys + OAuth2), RBAC, and the cross-cutting audit interceptor. After this phase the platform can be deployed via Docker, users can sign up, create an organisation, invite teammates with roles, and the OpenAPI document is auto-published — but no surveys exist yet.

### Tasks

#### 1.1 — Monorepo, Docker, and CI scaffold

**What**: Stand up the pnpm workspace, the NestJS API app, the Next.js web app, the Python analysis service stub, and a `docker-compose.yml` wiring Postgres, Redis, MinIO, and Mailpit.

**Design**:
- `docker-compose.yml` services: `postgres:17`, `redis:7`, `minio`, `mailpit`, `api` (Dockerfile.api), `web` (Dockerfile.web), `analysis` (Dockerfile.analysis).
- Environment via `.env` (validated by Zod `EnvSchema` at boot):
  ```
  DATABASE_URL=postgresql://srp:srp@postgres:5432/srp
  REDIS_URL=redis://redis:6379
  S3_ENDPOINT=http://minio:9000
  S3_BUCKET=srp
  JWT_SECRET=...            # min 32 bytes
  ANALYSIS_SERVICE_URL=http://analysis:8000
  ANTHROPIC_API_KEY=...
  AI_PROVIDER=anthropic     # anthropic | openai_compatible | disabled
  DATA_RESIDENCY=us         # default region label
  PUBLIC_BASE_URL=http://localhost:3000
  ```
- NestJS bootstraps `@nestjs/swagger` → serves OpenAPI 3.1 JSON at `GET /api/openapi.json` and Swagger UI at `/api/docs`.
- Python `analysis` service: FastAPI app exposing `GET /healthz` returning `{"status":"ok"}`.
- CI (GitHub Actions): `pnpm lint && pnpm typecheck && pnpm test`; `ruff check && mypy && pytest` for the analysis service; `docker build` for all three images.

**Testing**:
- `Unit: EnvSchema with all required vars → parses; missing JWT_SECRET → throws with field name "JWT_SECRET"`.
- `Integration (Testcontainers): boot api against ephemeral Postgres → GET /api/health returns 200 {status:"ok", db:"up", redis:"up"}`.
- `Integration: GET /api/openapi.json → valid OpenAPI 3.1 document (openapi field starts "3.1")`.
- `E2E: docker compose up → all healthchecks green within 60s`.

#### 1.2 — Tenancy & identity schema (Prisma)

**What**: Relational tables for organisations, users, memberships, workspaces — the stable core of the hybrid model (from data-model-suggestion-3 / -1).

**Design** (Prisma models; `@@map` to snake_case tables):
```prisma
model Organization {
  id                 String   @id @default(uuid()) @db.Uuid
  name               String
  slug               String   @unique
  planTier           PlanTier @default(free)
  billingEmail       String?
  dataResidency      Residency @default(us)
  gdprDpaSigned      Boolean  @default(false)
  maxResponsesMonth  Int      @default(1000)
  createdAt          DateTime @default(now())
  updatedAt          DateTime @updatedAt
  deletedAt          DateTime?
  memberships        OrganizationMembership[]
  workspaces         Workspace[]
}
enum PlanTier { free starter professional enterprise }
enum Residency { us eu ap au }

model User {
  id            String   @id @default(uuid()) @db.Uuid
  email         String   @unique
  emailVerified Boolean  @default(false)
  passwordHash  String?
  fullName      String
  locale        String   @default("en")
  timezone      String   @default("UTC")
  lastLoginAt   DateTime?
  createdAt     DateTime @default(now())
  deletedAt     DateTime?
  memberships   OrganizationMembership[]
}

model OrganizationMembership {
  id             String  @id @default(uuid()) @db.Uuid
  organizationId String  @db.Uuid
  userId         String  @db.Uuid
  role           OrgRole @default(member)
  invitedBy      String? @db.Uuid
  acceptedAt     DateTime?
  organization   Organization @relation(fields: [organizationId], references: [id], onDelete: Cascade)
  user           User         @relation(fields: [userId], references: [id], onDelete: Cascade)
  @@unique([organizationId, userId])
}
enum OrgRole { owner admin editor analyst member viewer }

model Workspace {
  id             String @id @default(uuid()) @db.Uuid
  organizationId String @db.Uuid
  name           String
  createdBy      String @db.Uuid
  organization   Organization @relation(fields: [organizationId], references: [id], onDelete: Cascade)
}
```

**Testing**:
- `Integration: create org + owner membership → unique(org,user) enforced; second membership for same pair → P2002 error`.
- `Integration: delete organization → memberships and workspaces cascade-deleted`.
- `Unit: slug generated from name "Acme Research Co" → "acme-research-co"; collision appends -2`.

#### 1.3 — Authentication, API keys, OAuth2

**What**: UI session auth (Auth.js, email+password and OAuth providers), personal API keys, and an OAuth 2.0 authorization-code flow for third-party apps — per RFC 6749/6750/7519.

**Design**:
- Endpoints:
  - `POST /api/auth/register` `{email, password, fullName}` → creates User + Organization + owner membership; sends verification email (queued).
  - `POST /api/auth/login` → sets session cookie (UI) and can mint a short-lived JWT.
  - `POST /api/orgs/:orgId/api-keys` (role ≥ admin) → returns `{id, prefix, secret}` once; stores only a hash (`argon2`). Header `Authorization: Bearer srp_<prefix>_<secret>`.
  - OAuth2: `GET /api/oauth/authorize`, `POST /api/oauth/token` (authorization_code + refresh_token grants). Access tokens are JWTs (RFC 7519) with `org`, `sub`, `scope` claims; resources protected by Bearer tokens (RFC 6750).
- `ApiKeyStrategy` and `JwtStrategy` Passport guards resolve `{ userId, organizationId, scopes }` onto the request.
- Password hashing: argon2id. Rate limiting via Redis (10 login attempts / 15 min / IP).

**Testing**:
- `Integration: register new email → 201, org+owner created, verification email enqueued`.
- `Integration: register duplicate email → 409`.
- `Integration: API request with valid Bearer key → 200; tampered key → 401; key for org A used on org B resource → 403`.
- `Integration: OAuth token exchange with valid code → access+refresh JWT; expired code → 400 invalid_grant`.
- `Unit: JWT contains org/sub/scope claims and exp ≤ 1h`.

#### 1.4 — RBAC guard, tenancy isolation, and audit interceptor

**What**: A `@Roles()` guard enforcing org/workspace roles, an interceptor that scopes every query to the caller's `organizationId`, and an audit interceptor writing an immutable `audit_log` row for every mutating request.

**Design**:
- `RolesGuard` reads `@Roles('admin','owner')` metadata and the membership role; denies with 403 otherwise.
- `TenancyInterceptor` injects `organizationId` into a request-scoped context; all repository methods take an explicit `orgId` arg (no implicit globals) to prevent cross-tenant leakage.
- `audit_log` table (relational, append-only, partitioned by quarter per suggestion-1):
  ```prisma
  model AuditLog {
    id             BigInt   @id @default(autoincrement())
    organizationId String   @db.Uuid
    userId         String?  @db.Uuid
    action         String   // "survey.create", "response.export", ...
    entityType     String
    entityId       String?  @db.Uuid
    oldValues      Json?
    newValues      Json?
    ipAddress      String?
    createdAt      DateTime @default(now())
    @@index([organizationId, createdAt])
  }
  ```
- `AuditInterceptor` runs after the handler on non-GET requests; serialises a diff and writes the row asynchronously (fire-and-forget into BullMQ to avoid blocking response).

**Testing**:
- `Unit: RolesGuard with member role on @Roles('admin') route → 403; admin → pass`.
- `Integration: viewer attempts POST /surveys → 403, no audit row, no survey created`.
- `Integration: editor creates survey → audit_log row {action:"survey.create", entityId}`.
- `Integration (tenancy): user in org A requests GET /surveys/:id where survey belongs to org B → 404 (not 403, to avoid leaking existence)`.

---

## Phase 2: Survey Definition Model & Builder API

### Purpose
Define the JSONB survey-definition contract — the heart of the hybrid data model — with a versioned Zod/JSON-Schema validator, and build the CRUD + versioning + publish API. After this phase a researcher (via API) can author a survey with pages, all MVP question types, answer options, matrix rows, and branching logic, then publish an immutable version. This is the foundation every later phase builds on.

### Tasks

#### 2.1 — SurveyDefinition JSON Schema (the JSONB contract)

**What**: A versioned Zod schema (`@srp/shared`) describing the entire survey structure stored in `surveys.definition` JSONB, validated on every write against JSON Schema Draft 2020-12 (per `standards.md`). Designed to map cleanly onto SurveyJS's model.

**Design**:
```typescript
export const QUESTION_TYPES = [
  'single_choice','multiple_choice','dropdown','likert','likert_matrix','nps',
  'text_short','text_long','text_email','text_number','ranking','rating',
  'slider','date','matrix_single','matrix_multiple','semantic_differential',
  'constant_sum','image_choice','file_upload',
  // advanced (Phase 8): 'conjoint_profile','maxdiff_set'
] as const;

export const AnswerOption = z.object({
  id: z.string().uuid(),
  label: z.string(),
  value: z.string(),
  sortOrder: z.number().int(),
  isExclusive: z.boolean().default(false),  // "None of the above"
  isAnchored: z.boolean().default(false),   // excluded from randomization
  score: z.number().nullable().optional(),
  imageUrl: z.string().url().nullable().optional(),
});

export const Question = z.object({
  id: z.string().uuid(),
  type: z.enum(QUESTION_TYPES),
  code: z.string().optional(),            // researcher code, e.g. "Q1"
  title: z.string(),
  description: z.string().optional(),
  isRequired: z.boolean().default(false),
  sortOrder: z.number().int(),
  randomizeOptions: z.boolean().default(false),
  allowOther: z.boolean().default(false),
  options: z.array(AnswerOption).default([]),
  matrixRows: z.array(z.object({ id: z.string().uuid(), label: z.string(), sortOrder: z.number().int() })).default([]),
  config: z.record(z.unknown()).default({}), // type-specific: {min,max,step} slider, {npsLabels}, etc.
}).superRefine(validateByType);          // e.g. choice types require ≥2 options

export const LogicRule = z.object({
  id: z.string().uuid(),
  sourceQuestionId: z.string().uuid(),
  ruleType: z.enum(['skip_to','show_question','hide_question','show_page','hide_page','end_survey']),
  targetQuestionId: z.string().uuid().optional(),
  targetPageId: z.string().uuid().optional(),
  operator: z.enum(['equals','not_equals','contains','greater_than','less_than','is_answered','is_not_answered','in_set']),
  value: z.string().optional(),
  values: z.array(z.string()).optional(),
  connector: z.enum(['AND','OR']).default('AND'),
  group: z.number().int().default(0),
});

export const SurveyPage = z.object({
  id: z.string().uuid(),
  title: z.string().optional(),
  sortOrder: z.number().int(),
  randomize: z.boolean().default(false),
  questions: z.array(Question),
});

export const SurveyDefinition = z.object({
  schemaVersion: z.literal(1),
  defaultLanguage: z.string().default('en'),
  settings: z.object({
    allowBack: z.boolean().default(true),
    showProgressBar: z.boolean().default(true),
    randomizeQuestions: z.boolean().default(false),
    anonymousResponses: z.boolean().default(false),
    thankYouMessage: z.string().optional(),
  }).default({}),
  pages: z.array(SurveyPage),
  logicRules: z.array(LogicRule).default([]),
  translations: z.record(z.string(), z.unknown()).default({}), // {fr: {...}, ...}
});
```
- `validateByType`: choice/dropdown/image_choice/ranking require `options.length ≥ 2`; matrix types require `matrixRows.length ≥ 1`; slider requires `config.min < config.max`; nps fixes a 0–10 scale.
- Cross-reference validation: every `LogicRule.sourceQuestionId`/`targetQuestionId` must resolve to an existing question; reject otherwise.
- A static export to JSON Schema Draft 2020-12 (`zod-to-json-schema`) is published at `GET /api/schemas/survey-definition.json`.

**Testing**:
- `Unit: valid full definition (all MVP types) → parses`.
- `Unit: single_choice with 1 option → ValidationError mentioning "options"`.
- `Unit: LogicRule referencing unknown questionId → ValidationError`.
- `Unit: slider config min ≥ max → ValidationError`.
- `Unit: exported JSON Schema validates the same fixtures via AJV`.

#### 2.2 — Survey & version persistence

**What**: `surveys` table (relational metadata + JSONB `definition`) and `survey_versions` (immutable snapshots taken at publish).

**Design**:
```prisma
model Survey {
  id            String   @id @default(uuid()) @db.Uuid
  organizationId String  @db.Uuid
  workspaceId   String?  @db.Uuid
  title         String
  internalName  String?
  status        SurveyStatus @default(draft)
  surveyType    String   @default("standard")
  definition    Json     // SurveyDefinition (validated by Zod before write)
  currentVersion Int     @default(0)
  completionRatePrediction Decimal? @db.Decimal(5,2)
  estimatedDuration Int?  // seconds
  startsAt      DateTime?
  closesAt      DateTime?
  responseLimit Int?
  createdBy     String   @db.Uuid
  publishedAt   DateTime?
  createdAt     DateTime @default(now())
  updatedAt     DateTime @updatedAt
  deletedAt     DateTime?
  versions      SurveyVersion[]
  @@index([organizationId, status])
}
enum SurveyStatus { draft scheduled active paused closed archived }

model SurveyVersion {
  id         String  @id @default(uuid()) @db.Uuid
  surveyId   String  @db.Uuid
  version    Int
  definition Json    // frozen snapshot at publish time
  publishedBy String @db.Uuid
  publishedAt DateTime @default(now())
  survey     Survey  @relation(fields: [surveyId], references: [id], onDelete: Cascade)
  @@unique([surveyId, version])
}
```
- Why versioning: per data-model-suggestion-2's key insight — researchers must know exactly which design produced each response. Every `Response` (Phase 3) stores `surveyVersion`.
- Endpoints (all OpenAPI-documented):
  - `POST /api/surveys` `{title, surveyType, definition?}` → draft.
  - `GET /api/surveys`, `GET /api/surveys/:id`.
  - `PATCH /api/surveys/:id` → revalidate `definition` with Zod; reject if `status != draft`.
  - `POST /api/surveys/:id/publish` → validate, freeze a `SurveyVersion`, increment `currentVersion`, set status `active`/`scheduled`.
  - `POST /api/surveys/:id/duplicate`, `DELETE /api/surveys/:id` (soft delete).

**Testing**:
- `Integration: create+patch draft → definition persisted; patch with invalid definition → 422 with Zod error path`.
- `Integration: patch a published (active) survey → 409`.
- `Integration: publish → SurveyVersion row created, currentVersion=1; publish again after edit-via-new-draft → version 2`.
- `Integration: duplicate → new survey, status draft, version 0, distinct id`.

#### 2.3 — Branching-logic evaluation engine

**What**: A pure function that, given a `SurveyDefinition` and the answers so far, returns the next page/question and visibility — used by both the runtime (Phase 3) and the builder's logic preview.

**Design**:
```typescript
interface NavState { currentPageId: string; answers: Record<string, AnswerValue>; }
interface NavResult { visibleQuestionIds: string[]; nextPageId: string | null; ended: boolean; }
function evaluateNavigation(def: SurveyDefinition, state: NavState): NavResult;
```
- Evaluates each `LogicRule` whose `sourceQuestionId` is answered; groups by `group` with AND/OR via `connector`; applies `skip_to`/`show`/`hide`/`end_survey`.
- Deterministic and side-effect-free → fully unit-testable; no DB access.

**Testing**:
- `Unit: "if Q1 == 5 skip to page 3" with Q1=5 → nextPageId=page3`.
- `Unit: show_question rule unmet → question excluded from visibleQuestionIds`.
- `Unit: end_survey rule met → ended=true`.
- `Unit: AND group (Q1=5 AND Q2=yes), only Q1 met → rule not applied`.
- `Unit: no rules → linear page order preserved`.

---

## Phase 3: Distribution & Response Collection Runtime

### Purpose
Make surveys answerable. Build distribution channels (web link, email, QR, embed), the public unauthenticated respondent runtime API, response/answer storage (relational responses + per-type answer payloads), and the SurveyJS-rendered respondent UI conforming to WCAG 2.2 AA. After this phase the full collect loop works end-to-end: publish → share link → respondent answers with branching → response stored.

### Tasks

#### 3.1 — Distribution channels & links

**What**: `distribution_channels` with per-channel config and unique public slugs; QR code generation.

**Design**:
```prisma
model DistributionChannel {
  id            String  @id @default(uuid()) @db.Uuid
  surveyId      String  @db.Uuid
  channelType   String  // web_link | email | sms | qr_code | in_app | api | embed
  name          String
  isActive      Boolean @default(true)
  uniqueSlug    String? @unique
  settings      Json    @default("{}")  // channel-specific config
  responseCount Int     @default(0)
  createdBy     String  @db.Uuid
  @@index([surveyId])
}
```
- `POST /api/surveys/:id/channels` → for `web_link`/`qr_code` generates a slug; `GET /api/channels/:id/qr.png` renders a QR pointing at `${PUBLIC_BASE_URL}/s/{slug}`.
- Channel must reject creation if survey is not published.

**Testing**:
- `Integration: create web_link channel on active survey → slug generated, resolvable`.
- `Integration: create channel on draft survey → 409`.
- `Integration: GET qr.png → image/png with the slug URL encoded`.

#### 3.2 — Response & answer storage (hybrid)

**What**: Relational `responses` (queryable status/geo/timing/weight) plus `response_answers` storing per-question-type payloads as validated JSONB — the hybrid model's answer side.

**Design**:
```prisma
model Response {
  id            String  @id @default(uuid()) @db.Uuid
  surveyId      String  @db.Uuid
  surveyVersion Int
  channelId     String? @db.Uuid
  respondentId  String? @db.Uuid     // panel link (Phase 5)
  externalUserId String?             // in-app
  sessionId     String?
  status        ResponseStatus @default(in_progress)
  language      String  @default("en")
  ipAddress     String?
  geoCountry    String?
  durationSeconds Int?
  weight        Decimal @default(1.0) @db.Decimal(10,6)
  isTest        Boolean @default(false)
  isAnonymized  Boolean @default(false)
  startedAt     DateTime @default(now())
  completedAt   DateTime?
  answers       ResponseAnswer[]
  @@index([surveyId, status])
  @@index([surveyId, completedAt])
}
enum ResponseStatus { in_progress completed abandoned screened_out quota_full disqualified }

model ResponseAnswer {
  id          String @id @default(uuid()) @db.Uuid
  responseId  String @db.Uuid
  questionId  String @db.Uuid
  questionType String
  payload     Json   // validated against per-type AnswerPayload schema
  numericValue Decimal? @db.Decimal(15,4) // extracted for aggregation/indexing (likert/nps/slider/rating)
  textValue   String?                     // extracted for open-text indexing
  durationMs  Int?
  response    Response @relation(fields: [responseId], references: [id], onDelete: Cascade)
  @@unique([responseId, questionId])
  @@index([questionId, numericValue])
}
```
- `AnswerPayload` Zod union by question type (in `@srp/shared`): e.g. `single_choice → {optionId}`, `multiple_choice → {optionIds: string[]}`, `likert/nps/slider/rating → {value: number}`, `text_* → {text: string}`, `ranking → {ranked:[{optionId,rank}]}`, `matrix → {rows:[{rowId,optionId,value?}]}`, `constant_sum → {alloc:[{optionId,value}]}`, `file_upload → {fileUrl}`.
- On write, the API extracts `numericValue`/`textValue` from the payload to enable SQL aggregation and full-text indexing without parsing JSONB (the hybrid optimisation).

**Testing**:
- `Unit: likert payload {value:4} → numericValue=4 extracted`.
- `Unit: multiple_choice payload validates optionIds against survey definition options`.
- `Integration: upsert answer twice for same (response,question) → single row, latest payload (respondent changed answer)`.

#### 3.3 — Public respondent runtime API

**What**: Unauthenticated endpoints that drive a survey session: start, fetch current page (post-branching), submit answers, complete — enforcing the frozen `SurveyVersion`.

**Design**:
- `POST /api/runtime/:slug/start` → resolves channel→survey→active version; creates `Response{status:in_progress, surveyVersion}`; returns `{responseId, definition (frozen version), navState}`.
- `POST /api/runtime/responses/:id/answers` `{questionId, payload, durationMs}` → validates payload against the frozen definition; upserts `ResponseAnswer`; returns next nav via `evaluateNavigation` (Phase 2.3).
- `POST /api/runtime/responses/:id/complete` → sets `completed`, `durationSeconds`; enqueues `response.completed` event (dashboard projection + webhooks).
- Abandonment: a scheduled BullMQ job marks `in_progress` responses with no activity > 30 min as `abandoned`.
- Anti-abuse: per-IP rate limit and optional honeypot; `is_test` flag for previews from the builder.

**Testing**:
- `Integration: start on active survey → response created with correct frozen version`.
- `Integration: start on closed/paused survey → 410 Gone`.
- `Integration: submit answer violating definition (option not in version) → 422`.
- `Integration: full flow with a skip rule → skipped page's questions never requested; complete → status completed, completedAt set`.
- `Integration: abandonment job → stale in_progress becomes abandoned`.

#### 3.4 — Respondent UI (SurveyJS) + WCAG 2.2 AA

**What**: Next.js public route `/s/[slug]` rendering the survey with SurveyJS `survey-react-ui`, driving the runtime API, supporting RTL and progress bar.

**Design**:
- Map `SurveyDefinition` → SurveyJS JSON model; SurveyJS handles rendering, client-side required-field validation, and progress.
- Server-side navigation authority remains the runtime API (client cannot bypass branching/quota).
- RTL layout when `translations[lang].isRtl`; language switcher when translations exist.
- Accessibility: labels bound to controls, fieldset/legend for groups, `aria-live` error region, visible focus, contrast ≥ 4.5:1 — verified by axe-core (per WCAG 2.2 AA / ISO 40500).

**Testing**:
- `E2E (Playwright): open /s/{slug}, answer all required, submit → "thank you" shown, response completed`.
- `E2E: leave required blank → inline error, cannot advance`.
- `E2E (axe-core): zero serious/critical violations on each page`.
- `E2E: RTL survey renders right-to-left`.

---

## Phase 4: Real-Time Dashboard, Cross-Tab & Export

### Purpose
Turn collected responses into insight. Build the realtime response dashboard (live counts, per-question aggregation), the cross-tabulation engine with period-over-period comparison, and exports (CSV, SPSS `.sav`) — completing the MVP feature set from `features.md`.

### Tasks

#### 4.1 — Realtime aggregation projection

**What**: Redis-backed live counters plus a periodically-materialised per-question results table, pushed to clients over WebSocket.

**Design**:
- On each `response.completed`/`answer.submitted` event (BullMQ), increment Redis counters (`survey:{id}:started|completed|byChannel|byCountry`) and update `question_results` (mirrors data-model-suggestion-2 Projection 3):
  ```prisma
  model QuestionResult {
    surveyId    String @db.Uuid
    questionId  String @db.Uuid
    questionType String
    totalAnswers Int   @default(0)
    optionCounts Json  @default("{}")     // {optionId: count}
    numericMean  Decimal? @db.Decimal(15,4)
    numericStdDev Decimal? @db.Decimal(15,4)
    npsScore     Decimal? @db.Decimal(5,2)
    npsBuckets   Json?    // {promoters,passives,detractors}
    recentTextSamples Json @default("[]")
    updatedAt    DateTime @updatedAt
    @@id([surveyId, questionId])
  }
  ```
- WebSocket gateway `/ws/surveys/:id` (auth-guarded) emits deltas; NPS computed as `%promoters − %detractors`.

**Testing**:
- `Integration: submit 10 NPS responses (mix) → npsScore matches %promoters−%detractors`.
- `Integration: completing a response increments Redis completed counter and updates QuestionResult`.
- `Integration (ws): subscribed client receives a delta within 1s of completion`.

#### 4.2 — Cross-tabulation engine

**What**: Compute a contingency table of one question against another, with optional weighting and filters, plus period-over-period comparison.

**Design**:
- `POST /api/surveys/:id/crosstab` `{rowQuestionId, colQuestionId, weighted?:bool, filters?, compareTo?:{from,to}}` → `{cells:[[..]], rowTotals, colTotals, base}`.
- For MVP, computed in PostgreSQL (group-by on extracted `numericValue`/`optionCounts`); chi-square significance delegates to the analysis service (Phase 7).
- Weighting multiplies cell counts by `response.weight`.

**Testing**:
- `Integration: crosstab gender×satisfaction over fixture responses → cell counts match hand-computed expected`.
- `Integration: weighted crosstab with weights {1.5,0.5} → weighted totals correct`.
- `Integration: compareTo previous period → both period matrices returned`.

#### 4.3 — Response export (CSV, SPSS)

**What**: Export completed responses as CSV and SPSS `.sav` for offline analysis (MVP must-have).

**Design**:
- `POST /api/surveys/:id/exports` `{format: csv|spss, includeIncomplete?, anonymize?}` → enqueues a job; returns `{exportId}`; `GET /api/exports/:id` → status + signed S3 URL when ready.
- CSV: one row per response, one column per question (codes as headers), labels-or-values switch. SPSS via `savReaderWriter`/`pyreadstat` (in the Python service) preserving variable + value labels and measurement level.
- `anonymize:true` drops IP/geo/PII and applies k-anonymity check on demographics.

**Testing**:
- `Integration: CSV export → header row = question codes, one data row per completed response`.
- `Integration: SPSS export → .sav readable by pyreadstat with value labels intact`.
- `Integration: anonymized export → no IP/email columns present`.

---

## Phase 5: Panel Management & Quotas

### Purpose
Add respondent recruitment and sampling control — a key differentiator over Typeform/Formbricks. Build panels, respondents with extensible demographics, quota enforcement during collection, and longitudinal multi-wave studies.

### Tasks

#### 5.1 — Panels & respondents

**What**: `panels` and `panel_respondents` (relational core demographics + JSONB `customAttributes` for extensible fields — hybrid model), with consent fields.

**Design**:
```prisma
model Panel {
  id String @id @default(uuid()) @db.Uuid
  organizationId String @db.Uuid
  name String
  panelType String @default("internal") // internal|external|mixed
  source String?
  memberCount Int @default(0)
  respondents PanelRespondent[]
}
model PanelRespondent {
  id String @id @default(uuid()) @db.Uuid
  panelId String @db.Uuid
  email String?
  phone String?
  externalId String?
  gender String?
  birthYear Int?
  country String?
  region String?
  language String?
  customAttributes Json @default("{}")   // extensible demographics
  consentGiven Boolean @default(false)
  consentGivenAt DateTime?
  optOut Boolean @default(false)
  qualityScore Decimal? @db.Decimal(5,2)
  totalSurveysSent Int @default(0)
  totalSurveysCompleted Int @default(0)
  panel Panel @relation(fields: [panelId], references: [id], onDelete: Cascade)
  @@index([panelId])
  @@index([country])
}
```
- Bulk import: `POST /api/panels/:id/respondents:import` (CSV) → validates, dedupes by email, records consent.
- Targeting query: filter respondents by core columns + `customAttributes` JSONB path (GIN-indexed).

**Testing**:
- `Integration: CSV import 100 rows, 5 dupes → 95 created, 5 skipped, memberCount=95`.
- `Integration: filter respondents customAttributes.segment="enterprise" → only matching returned`.
- `Integration: import without consent column when panel requires consent → rejected`.

#### 5.2 — Quotas & screening

**What**: Per-survey quotas with conditions that screen out or end the survey when full — enforced server-side in the runtime.

**Design**:
```prisma
model SurveyQuota {
  id String @id @default(uuid()) @db.Uuid
  surveyId String @db.Uuid
  name String
  targetCount Int
  currentCount Int @default(0)
  isActive Boolean @default(true)
  actionOnFull String @default("screen_out") // screen_out|end_survey|redirect
  conditions Json  // [{questionId, operator, value}]
}
```
- Runtime checks quotas after each answer; on a full matching quota, sets response `screened_out`/`quota_full` and returns the configured action. `currentCount` incremented atomically in Redis to avoid overshoot under concurrency.

**Testing**:
- `Integration: quota target 2 for gender=female; 3 female respondents → 3rd screened_out`.
- `Integration (concurrency): 5 simultaneous submissions against target 3 → exactly 3 admitted (Redis atomic incr)`.

#### 5.3 — Longitudinal studies & waves

**What**: Group surveys into waves tracking the same cohort over time (`features.md` underserved opportunity).

**Design**:
```prisma
model LongitudinalStudy { id String @id @default(uuid()) @db.Uuid; organizationId String @db.Uuid; name String; panelId String? @db.Uuid; waves StudyWave[] }
model StudyWave { id String @id @default(uuid()) @db.Uuid; studyId String @db.Uuid; surveyId String @db.Uuid; waveNumber Int; scheduledAt DateTime?; launchedAt DateTime?; @@unique([studyId, waveNumber]) }
```
- Respondent identity carried across waves via `respondentId`; a `GET /api/studies/:id/cohort` returns per-respondent answers across waves for trend analysis.

**Testing**:
- `Integration: create study with 2 waves; same respondent answers both → cohort query returns both keyed by respondent`.
- `Unit: wave numbering unique per study (P2002 on duplicate)`.

---

## Phase 6: AI Survey Assistant & Open-Text Analysis

### Purpose
Deliver the AI-native advantage. Build the AI question assistant (draft from a brief), completion-rate prediction, automated open-text theme analysis, and AI executive summaries — all behind a provider abstraction so self-hosters can swap models.

### Tasks

#### 6.1 — AI provider abstraction

**What**: An `AiProvider` interface with an Anthropic implementation and an OpenAI-compatible fallback, plus prompt-caching and cost/usage logging.

**Design**:
```typescript
interface AiProvider {
  generateJson<T>(opts: { system: string; user: string; schema: ZodSchema<T>; maxTokens?: number }): Promise<T>;
  generateText(opts: { system: string; user: string; maxTokens?: number }): Promise<string>;
  embed(texts: string[]): Promise<number[][]>;
}
```
- All AI work runs in BullMQ jobs (never inline in request path). Usage logged to `ai_usage` (tokens, cost, model) per org for billing.
- `AI_PROVIDER=disabled` cleanly degrades AI features to 501 with a clear message (self-host without keys).

**Testing**:
- `Unit (mocked): generateJson returns object validated by schema; malformed model output → retried once then throws`.
- `Integration: AI_PROVIDER=disabled → AI endpoints return 501`.

#### 6.2 — AI question assistant & completion-rate prediction

**What**: Generate a draft `SurveyDefinition` from a plain-text research objective; predict completion rate for a draft.

**Design**:
- `POST /api/surveys/ai/draft` `{objective, audience?, lengthTarget?}` → AI returns a `SurveyDefinition` (validated by the Phase 2.1 Zod schema — invalid drafts are regenerated). System prompt instructs balanced question mix, neutral wording, logical ordering.
- `POST /api/surveys/:id/predict-completion` → returns `{completionRate, riskFactors[]}` from heuristics (length, open-text count, matrix density) refined by the model; stored on `survey.completionRatePrediction`.

**Testing**:
- `Integration (mocked AI): draft from objective → valid SurveyDefinition that passes Zod`.
- `Integration (mocked AI): predict-completion → 0–100 value persisted with riskFactors`.
- `Unit: heuristic baseline penalises 60-question surveys vs 10-question`.

#### 6.3 — Open-text theme analysis

**What**: Cluster and quantify themes across open-text answers with sentiment, linking each theme to supporting responses (no manual coding).

**Design**:
```prisma
model TextAnalysisJob { id String @id @default(uuid()) @db.Uuid; surveyId String @db.Uuid; questionId String @db.Uuid; status String @default("pending"); modelVersion String?; responseCount Int @default(0); themes TextTheme[] }
model TextTheme { id String @id @default(uuid()) @db.Uuid; jobId String @db.Uuid; label String; sentiment String?; responseCount Int @default(0); percentage Decimal? @db.Decimal(5,2); confidence Decimal? @db.Decimal(5,4) }
```
- Job: embed open-text answers (`AiProvider.embed`), cluster, label each cluster + assign sentiment (positive/negative/neutral/mixed) via `generateJson`, store percentages, map answers→themes. `pgvector` stores embeddings (per suggestion-4) for semantic re-query.
- `POST /api/surveys/:id/questions/:qid/analyze-text` → enqueues job; `GET /api/text-jobs/:id` → themes with counts and example quotes.

**Testing**:
- `Integration (mocked AI): 200 fixture comments → ≥1 theme, percentages sum ≈ 100, each theme links ≥1 answer`.
- `Integration: analyze on a non-text question → 400`.
- `Integration: job lifecycle pending→processing→completed`.

#### 6.4 — AI executive summary reports

**What**: Generate a markdown executive summary combining quantitative results and qualitative themes.

**Design**:
- `POST /api/surveys/:id/reports/executive-summary` → job gathers `QuestionResult` aggregates + `TextTheme`s, prompts the model, stores `generated_reports{contentMarkdown, modelUsed, status}`.
- Prompt template injects: response count, completion rate, top NPS/CSAT figures, significant cross-tabs, and top themes; instructs an executive-ready structure (key findings, recommendations, caveats on sample size).

**Testing**:
- `Integration (mocked AI): generate summary → markdown report stored, references real figures from fixtures`.
- `Integration: summary with <30 responses → includes low-sample caveat (asserted via prompt-context flag)`.

---

## Phase 7: Statistical Analysis Engine (Python Service)

### Purpose
Provide the research-grade quantitative depth that distinguishes the platform from SurveyMonkey/Typeform: significance testing, regression, ANOVA, and sample weighting — in the Python `analysis` service.

### Tasks

#### 7.1 — Analysis service contract & job orchestration

**What**: Define the FastAPI contract and the NestJS-side `analysis_runs`/`analysis_results` orchestration (from suggestion-1 §8).

**Design**:
```prisma
model AnalysisRun {
  id String @id @default(uuid()) @db.Uuid
  surveyId String @db.Uuid
  analysisType String // cross_tabulation|significance_test|regression|anova|sample_weighting
  parameters Json
  status String @default("pending")
  resultSummary Json?
  results AnalysisResult[]
}
model AnalysisResult { id String @id @default(uuid()) @db.Uuid; runId String @db.Uuid; key String; numericResult Decimal? @db.Decimal(20,10); pValue Decimal? @db.Decimal(10,8); ciLow Decimal?; ciHigh Decimal? }
```
- NestJS `POST /api/surveys/:id/analyses` enqueues a BullMQ job that POSTs a tidy data frame (extracted from `response_answers`) to the Python service `POST /analysis/{type}`; results persisted to `analysis_results`.
- Python request/response use Pydantic models; data passed as columnar JSON (questionCode→values + weights).

**Testing**:
- `Integration: enqueue significance_test → run row pending→completed with pValue stored`.
- `Unit (py): request schema validates; missing weight column defaults to 1.0`.

#### 7.2 — Statistical methods

**What**: Implement chi-square + Cramér's V, t-test/ANOVA, OLS/logistic regression, and sample weighting (raking/RIM).

**Design** (Python, `scipy`/`statsmodels`):
- `chi_square(rows, cols, weights)` → `{chi2, p_value, dof, cramers_v}`.
- `ttest/anova(groups, weights)` → `{statistic, p_value, dof, eta_squared}`.
- `regression(y, X, type)` → coefficients, std errors, p-values, R²/pseudo-R², CIs.
- `rake_weights(sample_margins, target_margins)` → per-respondent weights converging to population targets (iterative proportional fitting), written back to `response.weight`.
- All return CIs and p-values; methods are published academic techniques (no IP barriers per `features.md`).

**Testing**:
- `Unit (py): chi-square on a known 2×3 table → matches scipy.stats.chi2_contingency`.
- `Unit (py): OLS on synthetic y=2x+noise → slope ≈ 2 within CI`.
- `Unit (py): raking converges so weighted margins ≈ targets within 0.5%`.
- `Integration: significance test result surfaces in the crosstab API (Phase 4.2) p-value`.

---

## Phase 8: Advanced Research Methods — Conjoint & MaxDiff

### Purpose
Add choice-based conjoint and MaxDiff — the highest-end differentiators (Qualtrics/QuestionPro territory) at mid-market price. Includes experimental design generation, runtime presentation, and utility/score estimation.

### Tasks

#### 8.1 — Conjoint & MaxDiff design + storage

**What**: Design generation (attributes/levels → balanced choice tasks; items → MaxDiff sets) stored as JSONB on the survey definition with normalised response capture.

**Design**:
- Extend `SurveyDefinition` question types `conjoint_profile` and `maxdiff_set`; design config (attributes/levels, numTasks, profilesPerTask / items, numSets, itemsPerSet) lives in `question.config`.
- Balanced design generation (D-efficient for conjoint; near-orthogonal balanced incomplete block for MaxDiff) in the Python service: `POST /analysis/design/conjoint` and `/design/maxdiff` return the task/set plans, stored back on the definition at publish.
- Responses captured as `ConjointResponse{taskId, chosenProfileId}` / `MaxDiffResponse{setId, bestItemId, worstItemId}` (relational, from suggestion-1 §6).

**Testing**:
- `Unit (py): conjoint design with 3 attrs × 3 levels → balanced (each level appears ~equally), D-efficiency reported`.
- `Unit (py): maxdiff design → each item appears equally across sets`.
- `Integration: respondent completes conjoint task → ConjointResponse stored with chosen profile`.

#### 8.2 — Utility & score estimation

**What**: Estimate conjoint part-worth utilities (HB/MNL) and MaxDiff item scores.

**Design** (Python, `xlogit`/`statsmodels`):
- `conjoint_utilities(responses, design)` → per-attribute-level utilities + importance %, with simulation endpoint for share-of-preference.
- `maxdiff_scores(responses, design)` → per-item utility scores + rank.
- Results stored in `analysis_results`; exposed via `GET /api/surveys/:id/conjoint/utilities`.

**Testing**:
- `Unit (py): MNL on simulated choice data with known utilities → recovers ordering`.
- `Unit (py): maxdiff scoring → best-most/worst-least item ranks correctly on synthetic data`.
- `Integration: utilities endpoint returns importance % summing ≈ 100`.

---

## Phase 9: Experiments, Integrations & Reporting Outputs

### Purpose
Bridge surveys to causal inference (A/B/control), connect to the wider tool ecosystem (webhooks, CRM, Slack, Zapier), and produce shareable/branded outputs (PDF/PPT, live links, scheduled digests).

### Tasks

#### 9.1 — Experiment design (A/B + control)

**What**: Assign respondents to control/treatment groups and compare outcomes.

**Design** (from suggestion-1 §7):
```prisma
model Experiment { id String @id @default(uuid()) @db.Uuid; surveyId String @db.Uuid; name String; hypothesis String?; status String @default("draft"); allocationMethod String @default("random"); groups ExperimentGroup[] }
model ExperimentGroup { id String @id @default(uuid()) @db.Uuid; experimentId String @db.Uuid; groupName String; groupType String; allocationPct Decimal @db.Decimal(5,2); participantCount Int @default(0) }
model ExperimentAssignment { id String @id @default(uuid()) @db.Uuid; experimentId String @db.Uuid; groupId String @db.Uuid; responseId String @db.Uuid; @@unique([experimentId, responseId]) }
```
- Runtime assigns each new response to a group by `allocationPct` (deterministic hash for stable assignment); outcome comparison runs a significance test (Phase 7) between groups.

**Testing**:
- `Integration: 50/50 experiment over 1000 responses → groups within ±3% of target`.
- `Integration: outcome comparison returns p-value for treatment vs control on a target metric`.

#### 9.2 — Webhooks & integration hub

**What**: Outbound webhooks (HMAC-signed) and connectors for Salesforce/HubSpot/Slack + Zapier.

**Design**:
```prisma
model Webhook { id String @id @default(uuid()) @db.Uuid; organizationId String @db.Uuid; surveyId String? @db.Uuid; url String; secret String; events String[]; isActive Boolean @default(true); failureCount Int @default(0) }
```
- BullMQ delivers events (`response.completed`, `survey.closed`) with `X-SRP-Signature: sha256=...` (HMAC, matching Typeform's pattern in `standards.md`); exponential-backoff retry, auto-disable after N failures.
- Connectors implement a `Connector` interface (`onResponseCompleted`); credentials encrypted at rest (`config_encrypted`).

**Testing**:
- `Integration (mock receiver): response.completed → POST with valid HMAC signature; bad secret → signature mismatch detectable`.
- `Integration: failing endpoint → retried with backoff, disabled after threshold`.
- `Unit: Slack connector formats a completion notification`.

#### 9.3 — Reports: PDF/PPT, share links, scheduled digests

**What**: Branded PDF/PPT export of dashboards, public share links, and scheduled email digests.

**Design**:
- PDF via headless Chromium (Playwright) rendering a report template; PPT via `pptxgenjs`. Stored in S3; `POST /api/surveys/:id/reports/pdf` enqueues, returns signed URL.
- Public dashboard share link: `GET /d/{shareToken}` (read-only, no login, optional passcode) — mirrors Qualtrics/SurveyMonkey shareable dashboards.
- Scheduled digests: cron-driven BullMQ job emails a summary on a schedule per survey.

**Testing**:
- `Integration: PDF export → valid PDF in S3, contains survey title`.
- `Integration: share link → renders dashboard read-only without auth; revoked token → 404`.
- `Integration: scheduled digest job → email enqueued to subscribers with current figures`.

---

## Phase 10: Compliance, MCP Server & Production Hardening

### Purpose
Satisfy the regulatory and enterprise-readiness requirements from `standards.md`, expose the AI-agent surface, and harden for production/self-hosted deployment.

### Tasks

#### 10.1 — GDPR/CCPA tooling: consent ledger, erasure, export

**What**: Consent capture, an immutable consent ledger, and respondent data subject requests (erasure/export/rectification).

**Design** (from suggestion-1 §10):
```prisma
model ConsentRecord { id String @id @default(uuid()) @db.Uuid; respondentId String? @db.Uuid; responseId String? @db.Uuid; consentType String; consentGiven Boolean; consentText String; legalBasis String @default("consent"); givenAt DateTime @default(now()); withdrawnAt DateTime? }
model DataDeletionRequest { id String @id @default(uuid()) @db.Uuid; organizationId String @db.Uuid; requesterEmail String; respondentId String? @db.Uuid; requestType String; status String @default("pending"); processedAt DateTime? }
```
- `POST /api/compliance/requests` (erasure/export/rectification) → for erasure, a job anonymises/removes PII across `panel_respondents`, `responses` (IP/geo/external IDs), and `response_answers` flagged PII; export bundles the subject's data as JSON+CSV; all actions audited.
- Data residency: organisation `dataResidency` controls which regional storage bucket/DB connection is used (self-host: single region; cloud: per-region).

**Testing**:
- `Integration: erasure request → respondent PII nulled, responses anonymized, audit rows written, request marked completed`.
- `Integration: export request → bundle contains all of the subject's responses`.
- `Unit: consent withdrawal sets withdrawnAt and supersedes prior consent of same type`.

#### 10.2 — OWASP ASVS L2 hardening

**What**: Apply OWASP ASVS Level 2 controls across the API.

**Design**:
- Input validation everywhere (Zod DTOs), output encoding in the runtime UI, parameterised queries (Prisma), secrets via env/secret store, brute-force protection (Redis), security headers (CSP, HSTS), encrypted integration credentials, structured security logging, dependency scanning in CI.
- Survey runtime CSP forbids inline script except SurveyJS bundle hash.

**Testing**:
- `Integration: SQL-injection-style payload in answer text → stored as literal text, no error (parameterised)`.
- `Integration: missing/invalid CSP-violating script blocked (header asserted)`.
- `E2E: rate limit triggers 429 after threshold`.

#### 10.3 — MCP server

**What**: An MCP server exposing platform tools to AI research agents (differentiation per `standards.md`).

**Design** (`@modelcontextprotocol/sdk`):
- Tools: `create_survey(definition)`, `list_surveys()`, `get_responses(surveyId, filters)`, `run_analysis(surveyId, type, params)`, `analyze_open_text(surveyId, questionId)`, `generate_executive_summary(surveyId)`.
- Authenticates with an org-scoped API key; every tool call is RBAC-checked and audited exactly like the REST API (the MCP server calls the same service layer).

**Testing**:
- `Integration: MCP create_survey with valid definition → survey created, visible via REST`.
- `Integration: MCP run_analysis → analysis_run created and result returned`.
- `Integration: MCP call with read-only key on a write tool → denied`.

#### 10.4 — Production deployment & observability

**What**: Production docker-compose, Helm chart, healthchecks, metrics, and backup guidance.

**Design**:
- `docker-compose.prod.yml` + Helm chart (api, web, analysis, postgres, redis, minio, workers).
- `/metrics` Prometheus endpoint (request latency, queue depth, AI token usage); structured JSON logs with correlation IDs.
- DB migrations run as a pre-deploy job; quarterly partition creation automated (pg_cron) for `responses`, `audit_log`.
- Self-host quickstart in README: `docker compose up` → seeded admin.

**Testing**:
- `E2E: helm install on kind cluster → all pods ready, /api/health green`.
- `Integration: /metrics exposes queue depth and request counters`.
- `E2E: fresh docker compose up → register, build survey, collect a response, view dashboard (full smoke)`.

---

## Phase Summary & Dependencies

```
Phase 1: Foundation, Tenancy & Auth        ─── required by everything
    │
Phase 2: Survey Definition & Builder API   ─── requires 1
    │
Phase 3: Distribution & Collection Runtime ─── requires 2
    │
    ├── Phase 4: Dashboard, Cross-Tab, Export ─── requires 3  ┐ can parallel
    └── Phase 5: Panel Management & Quotas     ─── requires 3  ┘
         │
         ├── Phase 6: AI Assistant & Text Analysis ─── requires 4 (results) ┐
         ├── Phase 7: Statistical Engine (Python)   ─── requires 4          │ can parallel
         └── Phase 9: Experiments/Integrations/Reports ─ requires 4,5       ┘
              │
Phase 8: Conjoint & MaxDiff                ─── requires 7 (estimation) + 2 (definition)
    │
Phase 10: Compliance, MCP & Hardening      ─── requires 1–9 (wraps the whole API)
```

**Parallelism opportunities**
- Phases 4 and 5 can be built concurrently once Phase 3 lands.
- Phases 6, 7, and 9 can proceed in parallel after Phase 4 (Phase 9 also needs Phase 5).
- The Python `analysis` service (Phases 7 and 8) can be developed by a separate track against the documented FastAPI contract from the end of Phase 4.
- MCP server (10.3) and compliance (10.1) reuse the existing service layer and can be built incrementally as endpoints land.

---

## Definition of Done (per phase)

A phase is complete only when all of the following hold:

1. All tasks in the phase are implemented.
2. All unit and integration tests pass (`pnpm test`; `pytest` for the analysis service); Testcontainers-backed integration tests run against real Postgres/Redis.
3. Linting and formatting pass: `pnpm lint`, `prettier --check`; `ruff check` for Python.
4. Type checking passes: `tsc --noEmit`; `mypy` for Python.
5. Docker images build successfully (`docker build` for api/web/analysis) and `docker compose up` healthchecks are green.
6. The phase's feature works end-to-end (relevant Playwright E2E passes).
7. New config options are documented in `.env.example` and the README self-host section.
8. New/changed API endpoints appear in the auto-generated OpenAPI 3.1 document at `/api/openapi.json`.
9. Prisma migrations are created, reviewed, and apply cleanly on a fresh database.
10. For respondent-facing UI changes: axe-core scan reports zero serious/critical WCAG 2.2 AA violations.
11. For mutating endpoints: an `audit_log` entry is produced and verified.
```
