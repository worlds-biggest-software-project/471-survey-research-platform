# Survey & Research Platform — Feature & Functionality Survey

> Candidate #471 · Researched: 2026-05-02

## Solutions Analysed

| Tool | Type | Licence / Model | URL |
|------|------|-----------------|-----|
| Qualtrics | SaaS | Commercial (enterprise; SAP-owned) | https://www.qualtrics.com |
| SurveyMonkey / Momentive | SaaS | Commercial (free tier) | https://www.surveymonkey.com |
| Typeform | SaaS | Commercial (free tier) | https://www.typeform.com |
| QuestionPro | SaaS | Commercial (free tier) | https://www.questionpro.com |
| Formbricks | Open source / SaaS | AGPLv3 (self-host); commercial cloud | https://formbricks.com |

## Feature Analysis by Solution

### Qualtrics

**Core features**
- Full Experience Management (XM) suite spanning customer, employee, product, and brand research programmes
- 100+ question types including MaxDiff, conjoint analysis, card sort, and heatmap in addition to standard Likert and NPS formats
- Stats iQ: automated statistical analysis including regression, ANOVA, and text analysis without requiring data science expertise
- Panel management: audience sourcing, quota management, and incentive distribution through the Qualtrics platform
- Workflow automation: triggered actions based on survey responses including follow-up emails, case creation in Salesforce, and Jira ticket generation
- Predictive intelligence: ML-driven insight extraction from open-text and structured responses

**Differentiating features**
- Used by more than 13,000 organisations — largest installed base of any enterprise research platform
- XM Directory: unified contact and segment management linking survey data to CRM and behavioural records across programmes
- Text iQ: advanced NLP sentiment analysis and theme identification across open-text survey fields
- AI-generated survey design recommendations based on the stated research objective

**UX patterns**
- Survey builder with real-time preview and mobile optimisation checker
- Dashboard builder with executive and operational views shareable via link without login
- Response monitoring dashboard with live completion rate and demographic breakdown

**Integration points**
- Salesforce, SAP, Microsoft Dynamics, and HubSpot CRM integrations
- Marketo, Pardot, and Eloqua for survey-triggered marketing automation
- Snowflake, Redshift, and BigQuery data warehouse connectors
- 100+ Zapier-connected integrations

**Known gaps**
- Enterprise pricing makes it inaccessible for small research teams and independent researchers
- User interface complexity has a steep learning curve for occasional users
- Implementation and deployment often requires Qualtrics professional services engagement

**Licence / IP notes**
- Proprietary SaaS (SAP-owned). No open-source components. XM Directory and predictive intelligence are proprietary AI assets.

---

### SurveyMonkey / Momentive

**Core features**
- Accessible survey builder used by 250,000+ organisations and 40 million users; balance of ease-of-use and feature depth
- Question bank: 250+ pre-written, expert-reviewed questions by survey type (customer satisfaction, employee engagement, NPS)
- SurveyMonkey Audience: panel-based respondent recruitment with quota controls for quantitative research
- Auto-themes: visual analysis of open-text responses grouping similar answers automatically
- Benchmarks: comparison of own NPS and satisfaction scores against industry averages from SurveyMonkey's aggregate database

**Differentiating features**
- Market benchmark database is distinctive — few platforms can offer percentile comparison against industry peers
- HIPAA-compliant plan available for healthcare surveys — rare in this price segment
- Integrates with Salesforce, HubSpot, Mailchimp, Slack, and Microsoft Teams on standard plans

**UX patterns**
- Step-by-step survey builder requiring no training for basic survey creation
- Results dashboard with standard charts, word clouds, and cross-tabulation filters
- Collector management: multiple distribution channels (email, web link, QR code, SMS) tracked separately per survey

**Integration points**
- Salesforce and HubSpot for pushing survey responses to CRM contact records
- Slack and Microsoft Teams for survey distribution and result notifications
- Mailchimp for email list-based survey distribution

**Known gaps**
- Advanced statistical methods (conjoint, MaxDiff) are not available; advanced research designs require Qualtrics or QuestionPro
- Panel quality for SurveyMonkey Audience varies; not suitable for the most demanding quantitative research projects
- Export and API access require paid plans; free tier creates significant limitations for research teams

**Licence / IP notes**
- Proprietary SaaS. No open-source components.

---

### Typeform

**Core features**
- Conversational survey format: one-question-at-a-time presentation with animated transitions optimised for high completion rates
- Conditional logic: branching paths based on previous answers without technical configuration required
- Photo and video question support embedding rich media in the survey flow
- Conversational form builder AI: describes the survey objective in plain text and Typeform drafts the question flow automatically
- Integration with 200+ tools via native integrations and Zapier

**Differentiating features**
- Completion rate optimisation: the conversational format consistently outperforms traditional grid-based surveys on completion rate metrics
- Design flexibility: pixel-perfect brand customisation with custom fonts, colours, and backgrounds
- VideoAsk integration: video-based question delivery and response collection for qualitative research

**UX patterns**
- Builder with real-time form preview showing the respondent experience as questions are added
- Logic map visualiser showing the branching structure of the survey as a flowchart
- Analytics dashboard with drop-off rates per question identifying where respondents abandon the form

**Integration points**
- HubSpot, Salesforce, and Pipedrive CRM for response-to-contact data sync
- Google Sheets, Airtable, and Notion for lightweight response storage
- Slack for real-time response notifications

**Known gaps**
- Not designed for quantitative research; no statistical analysis engine, sampling controls, or panel management
- Large surveys (50+ questions) are not well-suited to the conversational format
- No data warehouse integration; data export is CSV or third-party connector only

**Licence / IP notes**
- Proprietary SaaS. No open-source components.

---

### Formbricks

**Core features**
- Open-source survey and feedback platform with self-hostable architecture
- In-app survey embedding: micro-surveys triggered within web and mobile applications based on user behaviour events
- Audience targeting: survey delivery segmented by user attributes, actions, and segments synced from the product database
- NPS, CSAT, CES, and custom response formats
- Analytics: response rate trends, NPS time series, and open-text theme extraction

**Differentiating features**
- Full data sovereignty via self-hosting under AGPLv3; all survey response data remains on the operator's own infrastructure
- In-app survey triggering based on product usage events — rare at this price point
- Link surveys and email surveys alongside in-app delivery from one platform

**UX patterns**
- No-code survey editor with mobile preview
- Targeting rules editor: define audience segments using user attribute conditions without code
- Response dashboard with filtering and CSV export for further analysis

**Integration points**
- REST API and webhooks for custom data pipeline integration
- Zapier for third-party tool connections
- Self-hosted deployment via Docker Compose or Kubernetes Helm chart

**Known gaps**
- Statistical analysis capabilities are minimal; advanced research methods require external tools
- Panel management and respondent recruitment are not supported
- Self-hosted deployment requires DevOps capability to maintain and upgrade

**Licence / IP notes**
- AGPLv3: modifications to Formbricks source files must be disclosed when the software is offered as a network service. The self-hosted community edition is free; cloud-managed version is commercial.

---

## Cross-Cutting Feature Themes

### Table-Stakes Features
- Drag-and-drop survey builder with multiple question types (Likert, NPS, multiple choice, ranking, open text, matrix)
- Branching logic and skip patterns configurable without technical expertise
- Multi-channel distribution: email, web link, QR code, SMS, and in-product intercept
- Real-time response dashboard with basic filtering and cross-tabulation
- PDF and spreadsheet export for offline analysis
- GDPR-compliant data handling with response anonymisation controls

### Differentiating Features
- Advanced statistical methods: conjoint analysis, MaxDiff, and significance testing for quantitative research (Qualtrics, QuestionPro)
- AI question assistant generating survey drafts from a plain-text research objective
- Industry benchmark comparison for NPS and satisfaction scores
- In-app survey triggering based on user behaviour events (Formbricks, Qualtrics)
- VideoAsk-style video response collection for qualitative research (Typeform)

### Underserved Areas / Opportunities
- Mid-market platform combining panel recruitment, conjoint/MaxDiff analysis, and AI text analysis in one product below Qualtrics pricing
- Longitudinal panel management: tracking the same respondent cohort across multiple survey waves over time
- Modular per-response pricing reducing barriers for project-based researchers who need capacity bursts
- Built-in experiment design linking survey questions to A/B test conditions for causal inference

### AI-Augmentation Candidates
- AI survey design: generating question wording, order, and branching logic from a research brief
- Automated open-text analysis: grouping and quantifying themes across open-ended responses without manual coding
- Completion rate prediction: estimating drop-off risk for each survey design before launch
- Synthesis reports: AI-generated executive summary of survey findings combining quantitative results and qualitative themes

## Legal & IP Summary

Formbricks (AGPLv3) is the only significant open-source option; modifications must be disclosed if offered as a network service. Commercial platforms (Qualtrics, SurveyMonkey, Typeform) are proprietary. Standard survey question formats (Likert scale, NPS, semantic differential) are academic conventions with no IP encumbrances. Statistical methods used in analysis engines (regression, ANOVA, conjoint analysis) are published academic techniques freely implementable. NPS (Net Promoter Score) was trademarked by Bain & Company and Satmetrix; the term can be used descriptively but is not freely licensable as a branded programme. Panel respondent recruitment involves purchasing data from survey panel providers (Lucid, Dynata, Cint) under commercial data licensing agreements. A new entrant building a survey platform using standard statistical libraries (R, Python scipy, statsmodels) faces no patent barriers.

## Recommended Feature Scope

**Must-have (MVP)**:
- Drag-and-drop survey builder: multiple choice, Likert, NPS, ranking, matrix, and open-text question types
- Branching logic and skip patterns configurable without code
- Multi-channel distribution: web link, email embed, and QR code
- Real-time response dashboard with cross-tabulation filters and period-over-period comparison
- Response export (CSV, SPSS) for offline statistical analysis
- GDPR-compliant response collection with anonymisation controls and data residency options

**Should-have (v1.1)**:
- AI question assistant generating survey drafts from a plain-text research objective
- Automated open-text theme analysis grouping and quantifying response themes
- In-app survey triggering based on user behaviour events for product teams
- Audience panel integration for respondent recruitment with quota controls
- Branded survey domain and full white-label customisation

**Nice-to-have (backlog)**:
- Advanced statistical methods: MaxDiff, conjoint analysis, and significance testing
- AI-generated executive summary reports from survey findings
- Longitudinal panel management for multi-wave research programmes
- Built-in experiment design linking survey cohorts to A/B test conditions
- Benchmark database for NPS and satisfaction score comparison against industry peers
