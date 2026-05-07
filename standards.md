# Standards & API Reference

> Project: Survey & Research Platform · Generated: 2026-05-07

## Industry Standards & Specifications

### ISO Standards

**ISO 20252:2019 — Market, Opinion and Social Research, including Insights and Data Analytics**
- URL: https://www.iso.org/standard/73671.html
- The primary international quality standard for organisations conducting survey-based market, opinion, and social research. Covers the full research lifecycle from initial client contact through to reporting. The third edition (2019) added normative annexes for six globally-recognised research methodologies (face-to-face, telephone, online, postal, qualitative, and passive data collection). Compliance with ISO 20252 is increasingly required by enterprise buyers and panel providers as a condition of accreditation.

**ISO/IEC 40500:2025 — Web Content Accessibility Guidelines (WCAG) 2.2**
- URL: https://www.iso.org/standard/58625.html
- ISO ratification of WCAG 2.2, covering the accessibility requirements for web-based content including forms, surveys, and questionnaires. A survey platform must meet at minimum WCAG 2.2 Level AA for keyboard access, sufficient colour contrast, appropriate labels on form controls, and effective error messaging to serve respondents with disabilities.

### W3C & IETF Standards

**WCAG 2.2 — Web Content Accessibility Guidelines**
- URL: https://www.w3.org/WAI/standards-guidelines/wcag/
- W3C Recommendation defining four principles (perceivable, operable, understandable, robust) for accessible web content. The W3C WAI Forms Tutorial (https://www.w3.org/WAI/tutorials/forms/) provides concrete implementation guidance for labelling, grouping, instructions, and error handling in multi-page forms directly applicable to survey builders.

**RFC 6749 — The OAuth 2.0 Authorization Framework**
- URL: https://datatracker.ietf.org/doc/html/rfc6749
- De-facto standard for delegated authorisation used by all major survey platforms to allow third-party applications to access survey data on behalf of a user. Required for building any public developer API with third-party OAuth apps (e.g., Zapier, Salesforce integrations).

**RFC 7519 — JSON Web Token (JWT)**
- URL: https://datatracker.ietf.org/doc/html/rfc7519
- Defines a compact, URL-safe means of representing claims transferred between parties as a JSON object. Used extensively in survey platform APIs for access tokens and, in the OpenID Connect User Questioning API context, for User Statement Tokens that encode respondent identity assertions.

**RFC 7643 / RFC 7644 — SCIM 2.0 (System for Cross-domain Identity Management)**
- URL: https://datatracker.ietf.org/doc/html/rfc7643
- SCIM 2.0 defines a standard schema and REST API for automated provisioning and de-provisioning of user accounts in enterprise applications. Qualtrics and SurveyMonkey Enterprise both expose SCIM 2.0 endpoints (e.g., `https://api.surveymonkey.com/scim/v2`) enabling corporate SSO systems to automatically create and manage survey platform accounts. Any enterprise-targeting survey platform should support SCIM 2.0 to meet IT buyer requirements.

**RFC 8288 — Web Linking**
- URL: https://datatracker.ietf.org/doc/html/rfc8288
- Defines the `Link` header field for RESTful APIs, enabling hypermedia-driven pagination in survey response collections (a common pattern in survey platform REST APIs).

**RFC 7235 / RFC 6750 — HTTP Authentication and Bearer Token Usage**
- URL: https://datatracker.ietf.org/doc/html/rfc6750
- Bearer token specification for protecting REST API resources. Used by all major survey REST APIs (SurveyMonkey, Typeform, Formbricks, Alchemer) for API authentication.

### Data Model & API Specifications

**OpenAPI Specification 3.1.x / 3.2.x**
- URL: https://spec.openapis.org/oas/v3.2.0.html
- The vendor-neutral, language-agnostic standard for describing RESTful APIs in YAML or JSON. OpenAPI 3.1.0 introduced full JSON Schema Draft 2020-12 compatibility. In 2026, OpenAPI has become the default API description format for any public or partner API, and AI agents increasingly rely on OpenAPI specifications as the machine-readable bridge for tool invocation. A survey platform's public API should be described with an OpenAPI 3.1+ document to maximise discoverability, SDK auto-generation, and AI-agent integration.

**JSON Schema (Draft 2020-12)**
- URL: https://json-schema.org/specification
- Widely used to define the structure and validation rules for survey and response JSON payloads. SurveyJS, Formbricks, and other platforms represent survey definitions as JSON objects validated against a schema, enabling programmatic generation and validation of survey models without UI tooling.

**DDI — Data Documentation Initiative (Codebook 2.5 / Lifecycle 3.3 / v4 beta)**
- URL: https://ddialliance.org/
- An international metadata standard (XML-based in versions 2 and 3; XML/RDF/JSON in v4) designed specifically for documenting socioeconomic surveys, censuses, and other microdata collection activities. DDI Codebook (v2.5) is widely used by academic and government statistical agencies for archival exchange of survey instruments and datasets. DDI Lifecycle (v3.3) covers the full research lifecycle. Relevant when building platforms targeting academic, government, or archival research use cases. DDI-Lifecycle 4.0 is in final beta review as of 2026.

**HL7 FHIR Questionnaire / QuestionnaireResponse Resources (v5.0.0 / v6.0.0-ballot)**
- URL: https://hl7.org/fhir/questionnaire.html
- HL7 FHIR defines standardised Questionnaire and QuestionnaireResponse resources for capturing structured survey data in healthcare systems. The Structured Data Capture (SDC) implementation guide extends FHIR for advanced survey capabilities including multi-language support, conditional logic, and calculated scores. Relevant if the platform targets clinical research, patient-reported outcomes, or health systems integration.

### Security & Authentication Standards

**OpenID Connect 1.0**
- URL: https://openid.net/developers/how-connect-works/
- Authentication layer on top of OAuth 2.0, standardising SSO, identity tokens (JWT), and UserInfo endpoint access. The OpenID Connect User Questioning API 1.0 (https://openid.net/specs/openid-connect-user-questioning-api-1_0.html) is a niche specification for capturing user consent as a verifiable statement token — directly applicable to survey-based consent collection workflows.

**OWASP Application Security Verification Standard (ASVS)**
- URL: https://owasp.org/www-project-application-security-verification-standard/
- OWASP ASVS Level 2 defines security requirements for applications handling sensitive data collected via surveys (PII, health, financial). Key controls include input validation, access control, secure credential storage, and logging. Relevant for platforms that handle GDPR/CCPA-regulated response data.

**GDPR (EU Regulation 2016/679) and CCPA/CPRA**
- URL: https://gdpr.eu/ and https://oag.ca.gov/privacy/ccpa
- The two dominant data-privacy regulatory frameworks governing survey platforms that collect personal data from EU and California residents respectively. Key requirements for survey platforms include: lawful basis documentation for data collection; granular consent capture and withdrawal mechanisms; data minimisation; respondent rights (access, erasure, portability); cross-border transfer safeguards; and data retention schedules. As of 2025, 20+ US states have enacted equivalent state privacy laws.

### MCP Server Specifications

**Model Context Protocol (MCP)**
- URL: https://modelcontextprotocol.io/
- Anthropic's open protocol for connecting AI assistants to external tools, data sources, and APIs. An MCP server for a survey platform would expose tools for creating surveys, retrieving responses, and triggering analysis pipelines — enabling LLM-powered research agents to interact directly with the platform. Given the growing role of AI agents in automating survey design and analysis, MCP integration represents a near-term differentiation opportunity.

---

## Similar Products — Developer Documentation & APIs

### SurveyMonkey / Momentive
- **Description:** The most widely adopted commercial survey platform (250,000+ organisations), offering REST API v3 for survey creation, distribution, response retrieval, and webhook management.
- **API Documentation:** https://api.surveymonkey.com/v3/docs
- **Developer Portal:** https://developer.surveymonkey.com/ (US); https://developer.eu.surveymonkey.com/ (EU)
- **SDKs/Libraries:** No official SDK; community Python libraries available; Postman collection provided.
- **Developer Guide:** https://help.surveymonkey.com/en/surveymonkey/integrations/surveymonkey-api/
- **Standards:** REST/JSON; OpenAPI description available via Postman; SCIM 2.0 on Enterprise tier.
- **Authentication:** OAuth 2.0 (Bearer token); API Key for private apps. Rate limits: 120 requests/minute, 500 requests/day on free tier.

### Qualtrics
- **Description:** Enterprise Experience Management (XM) suite used by 13,000+ organisations, with a comprehensive REST API v3 for survey management, response export, directory management, and workflow automation.
- **API Documentation:** https://api.qualtrics.com/
- **Developer Portal:** https://www.qualtrics.com/support/integrations/developer-portal/
- **SDKs/Libraries:** Python and Java code examples provided; Extension SDK for custom app development; Postman public workspace.
- **Developer Guide:** https://www.qualtrics.com/support/integrations/api-integration/overview/
- **Standards:** REST/JSON; proprietary API description (no public OpenAPI spec); SCIM 2.0 for enterprise user provisioning.
- **Authentication:** API Token via `X-API-TOKEN` header (not OAuth 2.0). Token management through account settings.

### Typeform
- **Description:** Conversational survey and form platform with a REST API for form creation, response retrieval, and webhook management; known for high-completion conversational UX.
- **API Documentation:** https://developer.typeform.com/
- **SDKs/Libraries:** Official JavaScript SDK — `@typeform/api-client` (npm/yarn); GitHub: https://github.com/Typeform/js-api-client
- **Developer Guide:** https://www.typeform.com/developers/get-started/applications/
- **Standards:** REST/JSON; webhook payloads signed with HMAC SHA256 for payload verification.
- **Authentication:** OAuth 2.0 (access token in Authorization header). Webhook retry policy with 30-second timeout.

### Formbricks
- **Description:** Open-source Qualtrics alternative (AGPLv3) with a dual-API architecture: a public Client API for frontend survey interactions (no authentication required) and a Management API for backend administration.
- **API Documentation:** https://formbricks.com/docs/api-reference/rest-api
- **Developer Documentation Overview:** https://formbricks.com/docs/developer-docs/overview
- **SDKs/Libraries:** JavaScript/TypeScript SDK for in-app survey triggering; REST API with Postman collection. GitHub: https://github.com/formbricks/formbricks
- **Developer Guide:** https://formbricks.com/docs/developer-docs/rest-api
- **Standards:** REST/JSON; AGPLv3 open-source licence; self-hostable on Docker.
- **Authentication:** Client API — no authentication (public); Management API — personal API key generated in settings.

### Alchemer (formerly SurveyGizmo)
- **Description:** Enterprise workflow-integration survey platform with a REST API v5 exposing survey, question, page, and response management; strong custom scripting capabilities for complex branching logic.
- **API Documentation:** https://apihelp.alchemer.com/help
- **Developer Home:** https://developer.alchemer.com/help
- **SDKs/Libraries:** No official SDK; documented community PHP and Python examples.
- **Developer Guide:** https://help.alchemer.com/help/api
- **Standards:** REST/JSON; API versioned at v5; custom scripting engine (Alchemer Script).
- **Authentication:** API Key + Secret Key passed as query parameters or headers.

### QuestionPro
- **Description:** Research-professional-oriented platform with advanced methods (MaxDiff, conjoint analysis) and a REST API for survey management, panel management, and response retrieval.
- **API Documentation:** https://www.questionpro.com/api/index.html
- **Getting Started:** https://www.questionpro.com/api/getting-started.html
- **SDKs/Libraries:** No official SDK; REST API with HTTP authentication.
- **Standards:** REST/JSON; built-in HTTP authentication and HTTP verbs.
- **Authentication:** HTTP Basic Authentication / API Key.

### LimeSurvey
- **Description:** Open-source survey platform (GPLv2) with a JSON-RPC / XML-RPC Remote Control 2 API (LSRC2) for external programmatic control of surveys, participants, and responses.
- **API Documentation:** https://www.limesurvey.org/manual/RemoteControl_2_API
- **REST API:** https://www.limesurvey.org/manual/REST_API
- **SDKs/Libraries:** Python client `citric` (https://github.com/edgarrmondragon/citric); PHP and Python community libraries.
- **Developer Guide:** https://manual.limesurvey.org/RemoteControl
- **Standards:** JSON-RPC / XML-RPC (LSRC2); REST API added in recent versions. GPLv2 open-source.
- **Authentication:** Session key obtained via `get_session_key` function call; token passed with subsequent requests.

### SurveyJS
- **Description:** Open-source JavaScript form builder library (MIT licence) for React, Angular, Vue 3, and Vanilla JS. Provides a client-side survey rendering engine driven by JSON schema definitions, a drag-and-drop form creator, dashboard visualisation, and PDF export. Designed to be embedded in developer-built applications with any backend.
- **API Documentation:** https://surveyjs.io/documentation
- **GitHub:** https://github.com/surveyjs/survey-library (form library); https://github.com/surveyjs/survey-creator (form designer)
- **SDKs/Libraries:** npm packages: `survey-core`, `survey-react-ui`, `survey-angular-ui`, `survey-vue3-ui`.
- **Developer Guide:** https://surveyjs.io/form-library/documentation
- **Standards:** JSON Schema for survey model definitions; MIT licence; framework-agnostic backend integration.
- **Authentication:** No built-in authentication layer — integrates with application-level auth.

---

## Notes

**Emerging: AI-to-Survey Agent Protocols.** The intersection of LLM-based research agents and survey platforms is nascent but fast-moving. OpenAPI specifications are increasingly used as AI tool definitions, and MCP server adapters are beginning to appear for SaaS platforms. A survey platform that publishes an MCP server alongside a well-documented OpenAPI spec will be significantly easier to integrate into AI-driven research pipelines.

**No single canonical survey exchange format.** DDI serves academic/government archival use cases; FHIR Questionnaire serves healthcare; SurveyJS and Formbricks use their own proprietary JSON schemas. An opportunity exists to define (or adopt) a lightweight, open interchange format for survey definitions that bridges research platforms, analytics tools, and AI agents.

**Privacy regulation fragmentation.** GDPR and CCPA/CPRA are the dominant frameworks, but the proliferation of US state privacy laws (20+ as of 2025) and emerging frameworks in APAC and LATAM create compliance complexity. Survey platforms increasingly need to support jurisdiction-aware consent capture, data residency controls, and automated data deletion workflows.
