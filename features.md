# Succession Planning Tool — Feature & Functionality Survey

> Candidate #86 · Researched: 2026-05-01

## Solutions Analysed

| Tool | Type | Licence / Model | URL |
|------|------|-----------------|-----|
| SAP SuccessFactors Succession & Development | Commercial SaaS | Proprietary; PEPM subscription | https://www.sap.com/products/hcm/succession-development.html |
| Oracle HCM Cloud — Talent Management | Commercial SaaS | Proprietary; PEPM subscription | https://www.oracle.com/human-capital-management/talent-management/ |
| Workday Succession Planning | Commercial SaaS | Proprietary; bundled add-on | https://www.workday.com/en-us/products/human-capital-management/talent-management.html |
| Cornerstone OnDemand | Commercial SaaS | Proprietary; PEPM subscription | https://www.cornerstoneondemand.com/ |
| Eightfold AI | Commercial SaaS | Proprietary; custom enterprise | https://eightfold.ai/ |
| TalentGuard | Commercial SaaS | Proprietary; PEPM subscription | https://www.talentguard.com/ |
| emPerform | Commercial SaaS | Proprietary; PEPM subscription | https://www.employee-performance.com/ |
| Trakstar Perform | Commercial SaaS | Proprietary; annual subscription | https://www.trakstar.com/ |

## Feature Analysis by Solution

### SAP SuccessFactors Succession & Development

**Core features**
- Talent pools: configurable talent pools by role family, leadership tier, business unit, or custom criteria with automated candidate matching
- 9-box grid calibration: performance-versus-potential matrix with facilitated calibration session workflows; supports manager and HR consensus rating
- Nomination workflows: structured process for managers and HR business partners to nominate successors for key positions with approval routing
- Readiness assessment: multi-level readiness ratings (Ready Now, 1–2 Years, 3–5 Years) linked to development plans and learning assignments
- Succession org chart: visual hierarchy of positions with successor bench depth indicators and risk concentration heatmaps
- AI-driven successor matching: 1H 2026 update introduces AI recommendations for best-fit successors based on skills, performance history, and career trajectory

**Differentiating features**
- Market leadership with 12.6% market share; deepest integration with SAP HCM, payroll, and learning
- Calibration facilitation: structured group calibration workflows with in-session consensus tracking — not just individual ratings
- SAP Business AI integration: natural language performance summaries and AI-generated successor shortlists in the 1H 2026 update
- Global readiness: multilingual, multi-currency, and multi-legal-entity succession planning for complex multinationals

**UX patterns**
- HRBP-facing succession org chart as primary navigation surface
- Manager-facing team succession dashboard showing bench strength and development gaps
- Calibration session UI designed for HR-facilitated group review sessions
- Executive dashboard with board-level succession risk summary

**Integration points**
- Native SAP HCM, SAP S/4HANA payroll, and SuccessFactors Performance & Goals
- SAP Learning Management System for development plan execution
- HR Open Standards API for external HRIS and ATS connections
- SAP Analytics Cloud for advanced succession analytics and reporting

**Known gaps**
- Complex configuration: implementations routinely cost $100K–$2M+ and take 6–18 months
- SAP ecosystem dependency: full value only realised when the organisation uses SAP payroll and learning
- AI features are new (1H 2026) and less mature than Eightfold's skills-based recommendations
- Expensive for mid-market: $8–$12 PEPM plus implementation costs exclude organisations under 500 employees

**Licence / IP notes**
- Fully proprietary; SAP AG; publicly traded (NYSE: SAP)
- No OSS components; AI model is SAP Business AI proprietary IP
- No patent issues identified for core succession planning features; implementation methodology may be proprietary

---

### Workday Succession Planning

**Core features**
- Succession bench management: visual bench depth indicators per position showing Ready Now, 1–2 Year, and 3–5 Year successors
- Skills cloud integration: successor matching uses Workday's ML-based skills ontology to identify candidates with relevant skills adjacencies
- Talent review facilitation: structured talent review meeting support with calibration tools embedded in Workday
- Succession risk concentration: dashboard highlighting positions with single successors or no bench coverage
- Career pathing integration: employee-facing career exploration linked to succession pools — transparency to employees about development paths
- Real-time analytics: succession metrics update continuously as employees complete development milestones, change roles, or receive new performance ratings

**Differentiating features**
- Zero ETL: for Workday HCM customers, succession draws directly from live performance, skills, and org data — no data integration required
- Excellent UX: Workday's design language makes succession tools more accessible to HR Business Partners than SAP or Oracle
- Skills Cloud: ML-inferred skills from job history and learning completions used in successor identification without manual skills entry

**UX patterns**
- Workday-native UX: consistent with the broader Workday HRIS interface; low adoption friction for existing Workday users
- Role-based views: CHRO board-level risk view, HRBP facilitation view, and employee career exploration view in one platform
- Mobile-accessible for manager reviews

**Integration points**
- Native Workday HCM, Workday Learning, and Workday Peakon (employee listening)
- Workday Extend for custom integrations to non-Workday systems
- Limited connectors to non-Workday HRIS systems

**Known gaps**
- Low standalone value: buying Workday succession without Workday HCM is effectively impossible
- Skills Cloud is proprietary and opaque; organisations cannot export the inferred skills graph
- Succession planning is a secondary module; depth of calibration tooling lags SAP SuccessFactors for complex enterprises

**Licence / IP notes**
- Fully proprietary; Workday Inc. (NASDAQ: WDAY)
- Skills Cloud AI model is proprietary Workday IP; no OSS components

---

### Eightfold AI

**Core features**
- Skills-based successor identification: AI scans the full workforce to identify best-fit successors based on current skills, skills trajectory, and career progression patterns — without requiring pre-nominated candidates
- Bias mitigation: succession recommendations include diversity signals and configurable equity lenses to identify underrepresented candidates in succession pools
- Readiness scoring: continuous readiness assessment updated from live signals (project participation, learning completions, performance data) rather than point-in-time ratings
- Internal mobility integration: successor identification feeds directly into open internal role recommendations for employees
- DE&I funnel reporting: tracks representation across the succession pipeline from identification through appointment

**Differentiating features**
- Most sophisticated AI successor identification in market: scans all employees rather than only pre-nominated candidates; surfaces hidden high-potentials
- Continuous rather than annual: readiness scores update in real time from live workforce signals
- Proactive gap identification: flags emerging succession gaps 6–12 months before they become critical

**UX patterns**
- Analytics-first: succession insights presented as dashboards rather than traditional org chart interfaces
- CHRO and talent strategy audience; less suited to manager-level succession reviews
- Integration-delivered: features accessed via Eightfold's talent intelligence platform rather than a standalone succession module

**Integration points**
- Deep HRIS integrations: Workday, SAP SuccessFactors, Oracle HCM
- ATS integrations: Greenhouse, Lever, iCIMS for external successor pipeline view
- LMS integrations: Cornerstone, Degreed, LinkedIn Learning for development plan execution

**Known gaps**
- Very expensive: $150K–$500K+/year; effectively limited to large enterprises
- Data-hungry: accuracy degrades significantly in organisations with fewer than 500 employees or poor data quality
- Traditional HR processes (9-box calibration sessions, nomination workflows) are secondary to AI insights
- Succession is one use case within a broader talent intelligence platform; dedicated succession features are less deep than SAP or Workday

**Licence / IP notes**
- Proprietary; $397M total funding; valued at $2.1B (2021)
- AI skills inference model is proprietary IP; patent portfolio covers talent matching methodology
- No OSS components

---

### TalentGuard

**Core features**
- Succession planning software: position-based succession plans with multi-tier candidate benches and readiness ratings
- Role readiness scoring: proprietary readiness algorithm combining performance ratings, competency assessments, and experience factors
- 9-box grid: configurable performance-versus-potential matrix with calibration session support
- Talent pools: configurable talent pools with automated candidate matching based on role competencies
- Bench strength dashboard: manager view of team succession coverage with tenure risk and gap identification
- Equity lens: configurable diversity filters for succession pipelines with representation gap reporting

**Differentiating features**
- Purpose-built succession and career pathing platform: faster time-to-value than HCM suite modules
- WorkforceGPT: AI-powered role recommendations and succession candidate identification using natural language
- Configurable equity lenses: surface diversity representation gaps in succession pools with configurable demographic filters
- Risk mitigation centre: identifies flight risks and succession concentration risks in one dashboard

**UX patterns**
- HRBP-facing succession dashboard with position-based and person-based views
- Manager team view showing bench strength, tenure risks, and development gaps
- Executive succession risk report for board-level presentation

**Integration points**
- HRIS connectors: Workday, SAP SuccessFactors, ADP, BambooHR
- LMS connectors: Cornerstone, Saba for development plan execution
- REST API for custom HRIS and ATS integrations

**Known gaps**
- Smaller ecosystem than SAP, Oracle, and Workday; fewer certified integration partners
- Limited payroll and compensation integration
- AI features (WorkforceGPT) are newer and less proven than Eightfold's established model
- No self-hosted or on-premises option

**Licence / IP notes**
- Proprietary SaaS; private company; no disclosed funding
- No OSS components

---

## Cross-Cutting Feature Themes

### Table-Stakes Features
- Position-based succession plans: identify key positions, designate successors, and assign readiness ratings
- 9-box grid: performance-versus-potential matrix for talent review calibration sessions
- Talent pools: configurable pools for leadership tiers, role families, or strategic programs
- Readiness assessment: structured readiness levels (Ready Now, 1–2 Years, 3–5 Years) with development plan linkage
- Bench strength reporting: dashboard showing positions with insufficient bench coverage or single-point succession risk
- Integration with HRIS performance and skills data for candidate identification

### Differentiating Features
- AI-powered successor identification across the full workforce (not just pre-nominated candidates) — Eightfold
- Continuous readiness scoring from live signals (not annual ratings) — Eightfold, TalentGuard
- Diversity and equity lens on succession pools with representation gap reporting — Eightfold, TalentGuard
- Scenario modeling: simulate bench strength impact under multiple simultaneous departure scenarios — no SMB/mid-market tool offers this without professional services
- Natural language AI assistant for succession queries and recommendations — TalentGuard WorkforceGPT, SAP Business AI (1H 2026)
- Career transparency for employees: employee-facing view of succession pools and development paths — Workday, Cornerstone

### Underserved Areas / Opportunities
- **Mid-market affordability**: all credible succession tools are either enterprise HCM modules (requiring $100K+ implementations) or SMB tools with shallow analytics; the 200–2,000 employee segment has no affordable, capable option
- **Open-source succession planning**: no OSS succession planning tool exists at any level of capability; the entire market is proprietary
- **Bench strength scenario modeling for mid-market**: enterprise tools offer professional-service-led scenario analysis; mid-market tools do not offer scenario modeling at all
- **Bias audit for succession decisions**: no tool provides an independent, auditable bias assessment of succession decisions made using AI recommendations
- **Board-ready reporting without professional services**: generating board succession risk reports is still a manual design task in most organisations

### AI-Augmentation Candidates
- AI successor identification across the full workforce: scan all employees for skills adjacency and career trajectory fit rather than starting from pre-nominated lists
- Continuous readiness scoring from passive signals: performance data, project participation, learning completions, and peer feedback — eliminating the annual calibration cycle dependency
- Bench strength scenario modeling: simulate simultaneous departures and identify minimum development investments to maintain coverage
- Natural language succession queries for HR leaders ("Who is ready now for the VP Finance role if current incumbent leaves?")
- Automated board succession risk narrative generation with visualisations

---

## Legal & IP Summary

- No OSS succession planning tools identified; the entire market is proprietary SaaS — there is no copyleft or licence compliance risk in creating a new OSS tool
- 9-box grid methodology is an industry-standard HR concept (originated at McKinsey in the 1970s) with no IP protection; free to implement
- SHRM succession planning framework is published guidance; freely usable as a reference
- Employee talent assessment data (performance ratings, potential assessments, succession nominations) is personal data under GDPR Article 4 and CCPA; processing requires documented lawful basis (typically legitimate interest for employment purposes) and robust access controls
- GDPR Article 22 restricts solely automated succession decisions with significant employment effects; human review must be a documented step in any AI-driven succession process
- Eightfold AI holds patents on skills inference and talent matching methodology; any OSS successor identification using AI must use an independent algorithmic approach (skills adjacency graphs using open taxonomies such as ESCO and O*NET are a viable alternative)
- SOC 2 Type II is expected by enterprise buyers for any cloud succession planning tool handling employee assessment data
- HR Open Standards (HR-XML) is an open standard for succession data exchange; freely implementable

---

## Recommended Feature Scope

**Must-have (MVP)**
- Position-based succession plans: designate key positions, add successors, assign readiness tiers (Ready Now / 1–2 Years / 3–5 Years)
- 9-box grid: configurable performance-versus-potential matrix with collaborative annotation for calibration sessions
- Talent pools: configurable pools with candidate management, notes, and development plan association
- Bench strength dashboard: position-level coverage heat map with single-point succession risk alerts
- HRIS integration via REST API: initially targeting Workday, BambooHR, and ADP for employee and performance data import
- Role-based access: CHRO, HRBP, manager, and executive views with appropriate data visibility controls
- Self-hosted deployment option for data-sovereign organisations

**Should-have (v1.1)**
- AI successor identification: scan the full workforce for candidate fit using skills adjacency matching against ESCO/O*NET taxonomies (independent of Eightfold's proprietary model)
- Diversity and equity lens: configurable representation filters on succession pools with gap reporting
- Bench strength scenario modeling: simulate simultaneous departure scenarios and calculate bench coverage impact
- Natural language succession queries ("Who covers the CFO role if Sarah leaves?")
- Board-ready succession risk report: auto-generated narrative with heat map visualisations exportable to PDF and PowerPoint

**Nice-to-have (backlog)**
- Employee-facing career transparency portal: employees see which talent pools they are in and what development is recommended for advancement
- Continuous readiness scoring from HRIS signals without annual rating cycle dependency
- Integration with LMS platforms to automatically assign learning based on identified succession gaps
- SHRM succession framework audit: automated assessment of whether the organisation's succession program meets SHRM best-practice criteria
- ISO 30414 succession metrics export for human capital disclosure reporting
