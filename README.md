# Survey & Research Platform

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An AI-native survey platform that combines survey design, panel management, and advanced statistical analysis at a price point accessible to mid-market research teams.

Survey & Research Platform is a full-lifecycle research tool for organisations that need more than a basic form builder but cannot justify the cost or complexity of enterprise experience-management suites. It targets product teams, market researchers, and operations leads who need to design surveys, recruit respondents, run statistical analyses, and share findings -- all without stitching together multiple tools or engaging professional services.

---

## Why Survey & Research Platform?

- **Enterprise tools are priced out of reach.** Qualtrics, the dominant enterprise platform, requires professional services engagements and enterprise contracts that exclude small research teams and independent researchers entirely.
- **Mid-market tools lack analytical depth.** SurveyMonkey does not offer advanced statistical methods such as conjoint analysis, MaxDiff, or significance testing -- forcing quantitative researchers to export data and analyse it elsewhere.
- **Conversational tools ignore research rigour.** Typeform optimises for completion rate and design polish but provides no statistical analysis engine, sampling controls, or panel management.
- **Open-source options are incomplete.** Formbricks provides self-hosted data sovereignty but has minimal statistical analysis capabilities and no respondent recruitment support.
- **AI text analysis still requires a data science team.** Incumbents either lack open-text theme extraction or lock it behind premium tiers, leaving most teams to manually code qualitative responses.

---

## Key Features

### Survey Design & Builder

- Drag-and-drop survey builder supporting multiple choice, Likert, NPS, ranking, matrix, and open-text question types
- Branching logic and skip patterns configurable without technical expertise
- AI question assistant that generates survey drafts from a plain-text research objective
- Completion rate prediction estimating drop-off risk for each survey design before launch
- Multilingual support with machine-translation assistance and right-to-left layout

### Distribution & Panel Management

- Multi-channel distribution: email, SMS, web link, QR code, in-product intercepts, and API triggers
- Audience panel integration for respondent recruitment with quota controls
- In-app survey triggering based on user behaviour events for product teams
- Longitudinal panel management for tracking respondent cohorts across multiple survey waves

### Analysis & Reporting

- Real-time analytics dashboard with cross-tabulation, sentiment analysis, and trend comparisons
- Statistical analysis engine with significance testing, regression, conjoint analysis (MaxDiff), and sample weighting
- Automated open-text theme analysis grouping and quantifying response themes without manual coding
- AI-generated executive summary reports combining quantitative results and qualitative themes
- Branded PDF/PPT exports, shareable live dashboard links, and scheduled digest emails

### Experiment Design

- Built-in A/B test and control group management bridging survey feedback and causal inference
- Modular per-response pricing supporting capacity bursts for project-based researchers

### Compliance & Collaboration

- GDPR and CCPA compliance tooling with response anonymisation controls and data residency options
- Role-based access, team workspaces, and full audit trail
- Integration hub with connectors to Salesforce, HubSpot, Slack, Tableau, and 200+ tools via Zapier and native APIs

---

## AI-Native Advantage

AI is embedded across the entire research lifecycle rather than bolted on as an upsell. The platform generates survey question wording, sequencing, and branching logic from a plain-text research brief, then predicts completion rates before launch so researchers can optimise design iteratively. After collection, automated open-text analysis groups and quantifies themes across thousands of free-text responses -- work that traditionally requires manual coding or a dedicated data science team. Synthesis reports combine quantitative and qualitative findings into executive summaries, reducing the turnaround from data collection to actionable insight.

---

## Tech Stack & Deployment

The platform targets self-hosted, cloud, and hybrid deployment modes. Self-hosted deployment via Docker Compose or Kubernetes is a first-class option for organisations requiring full data sovereignty. The statistical analysis engine can be built on standard open-source libraries (Python scipy, statsmodels, R) with no patent barriers. Standard survey question formats (Likert, NPS, semantic differential) are academic conventions with no IP encumbrances. Integration is exposed through a REST API and webhooks, with Zapier and native connectors for major CRM and data warehouse platforms.

---

## Market Context

The survey and research platform market is dominated by Qualtrics (13,000+ organisations, enterprise pricing) and SurveyMonkey (250,000+ organisations, 40 million users). Enterprise experience-management pricing puts advanced capabilities out of reach for mid-market teams, while accessible tools lack the statistical depth that quantitative research demands. The primary buyers are product managers, market research teams, HR/people operations, and customer experience programmes seeking a single platform that spans design, recruitment, analysis, and reporting without per-seat pricing barriers.

---

## Project Status

> This project is in the **research and specification phase**.  
> Contributions, feedback, and domain expertise are welcome.

---

## Contributing

We welcome contributions from developers, domain experts, and potential users.
See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

**Important:** All contributions must be your own original work or clearly attributed
open-source material with a compatible licence. Copyright infringement and licence
violations will not be tolerated and will result in immediate removal of the offending
contribution. If you are unsure whether a piece of code, text, or other material is
safe to contribute, open an issue and ask before submitting.

---

## Licence

Licence to be determined. See [discussion](#) for context.
