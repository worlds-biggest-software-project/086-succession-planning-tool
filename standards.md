# Standards & API Reference

> Project: Succession Planning Tool · Generated: 2026-05-06

## Industry Standards & Specifications

### ISO Standards

**ISO/TS 30433:2021 — Human Resource Management: Succession Planning Metrics Cluster**
- URL: https://www.iso.org/standard/68710.html
- The most directly relevant ISO standard for this project. Specifies the elements of succession planning metrics and provides comparable measures for internal and external reporting, including succession effectiveness rate, successor coverage rate, and succession readiness rate. Part of the ISO 30000 family and required for ISO 30414 compliance.

**ISO 30414:2025 — Human Resource Management: Requirements and Recommendations for Human Capital Reporting and Disclosure**
- URL: https://www.iso.org/standard/30414
- Second edition (August 2025), upgraded from guidelines to mandatory requirements across 58 metrics in 11 core areas. Succession planning is one of the 11 core areas; organisations seeking ISO 30414 compliance must export succession metrics in the formats defined by ISO/TS 30433. The 2025 edition significantly raises the bar for succession data quality and auditability.

**ISO 30400:2022 — Human Resource Management: Vocabulary**
- URL: https://www.iso.org/standard/78044.html
- Defines common HR terminology including succession-related vocabulary. Ensures consistent use of terms such as "talent pool", "succession plan", "readiness", and "bench strength" across systems and data exports. Second edition, published November 2022.

### W3C & IETF Standards

**RFC 7231 — HTTP/1.1 Semantics and Content (IETF)**
- URL: https://www.rfc-editor.org/rfc/rfc7231
- Governs HTTP semantics for all REST API communication. Relevant to the design and documentation of the tool's own REST API for HRIS data ingestion, succession data export, and third-party integrations.

**RFC 8288 — Web Linking (IETF)**
- URL: https://www.rfc-editor.org/rfc/rfc8288
- Defines link relations for RESTful hypermedia APIs. Relevant when designing pagination and related-resource navigation in the tool's succession data API.

**RFC 7617 — HTTP Basic Authentication (IETF)**
- URL: https://www.rfc-editor.org/rfc/rfc7617
- Foundational authentication spec relevant for simple API key implementations (e.g. BambooHR-style per-key authentication).

**RFC 6749 — OAuth 2.0 Authorization Framework (IETF)**
- URL: https://www.rfc-editor.org/rfc/rfc6749
- The authorisation standard used by Workday, SAP SuccessFactors, and ADP for their REST APIs. Any succession planning tool integrating with enterprise HRIS systems must implement OAuth 2.0 client credentials and authorisation code flows.

### Data Model & API Specifications

**OpenAPI Specification 3.1.0 / 3.2.0**
- URL: https://spec.openapis.org/oas/v3.1.0.html (3.1.0); https://spec.openapis.org/oas/v3.2.0.html (3.2.0)
- The standard for describing REST APIs in a machine-readable format. The succession planning tool's own API should be documented as an OpenAPI 3.1+ spec to enable SDK generation, automated testing, and third-party integration. OpenAPI 3.2.0 (September 2025) adds streaming-friendly media types and fresh OAuth flows.

**HR Open Standards (HR-XML / HR-JSON)**
- URL: https://www.hropenstandards.org/
- The primary open standard for HR data interchange. Provides XML and JSON schemas for employee records, talent assessments, succession nominations, and related data. HR Open Standards 4.1+ includes JSON schemas for assessment and recruiting data; the consortium is transitioning from XML to JSON-first APIs. Free to download and implement.

**SCIM 2.0 — System for Cross-domain Identity Management (RFC 7644)**
- URL: https://www.rfc-editor.org/rfc/rfc7644
- The standard for automated user provisioning and deprovisioning across applications. Relevant for enterprise deployments of the succession planning tool where user accounts should be automatically provisioned when employees join or leave the organisation via the connected HRIS.

**ESCO Classification API — European Skills, Competences, Qualifications and Occupations**
- URL: https://esco.ec.europa.eu/en/use-esco/use-esco-services-api
- A free, multilingual European Commission API providing the ESCO taxonomy of 3,039 occupations and 13,939 skills in 28 languages. Critical for AI-based successor identification using skills-adjacency matching (the recommended alternative to Eightfold's proprietary skills inference model). Freely usable in open-source tools.

**O*NET Web Services API — Occupational Information Network (US Department of Labor)**
- URL: https://services.onetcenter.org/
- US government API providing standardised occupational data, skills, and competency frameworks across 900+ occupations. The CareerOneStop "Get Skill Gaps Between Two Occupations" API (https://www.careeronestop.org/Developers/WebAPI/SkillsGaps/get-skills-gaps-between-two-occupations.aspx) is particularly relevant for bench-strength gap analysis. Free for non-commercial and open-source use.

### Security & Authentication Standards

**OAuth 2.0 (RFC 6749) and OpenID Connect**
- URL: https://openid.net/connect/
- OAuth 2.0 is the de facto authorisation standard for all major HRIS APIs (Workday, SAP, ADP). OpenID Connect (OIDC) adds identity layer on top of OAuth 2.0 for user authentication in multi-tenant SaaS deployments of the succession planning tool. Both are required for enterprise readiness.

**SCIM 2.0 (RFC 7644) for user provisioning**
- See Data Model section above. SCIM with OAuth 2.0 bearer tokens is the expected standard for enterprise HR SaaS user provisioning in 2025–2026.

**GDPR Article 22 — Automated Individual Decision-Making, Including Profiling**
- URL: https://gdpr-info.eu/art-22-gdpr/
- Restricts solely automated decisions that produce legal or similarly significant effects on employees. AI-generated succession recommendations must include a documented human review step; a rubber-stamp review does not satisfy Article 22. DPIA (Data Protection Impact Assessment) under Article 35 is required for any AI-driven succession scoring deployed in the EU.

**GDPR / CCPA — Data Subject Rights**
- GDPR Article 4 classifies talent assessments, potential ratings, and succession nominations as personal data. Employees have subject access rights to this data. CCPA applies equivalent rights to California employees. The tool must implement data subject access request (DSAR) workflows and data deletion capabilities.

**SOC 2 Type II (AICPA Trust Services Criteria)**
- URL: https://www.aicpa-cima.com/resources/landing/system-and-organization-controls-soc-suite-of-services
- De facto security compliance requirement for enterprise HR SaaS. Covers security, availability, processing integrity, confidentiality, and privacy. Required by enterprise buyers before procurement. Type II audit covers 6–12 months of operating effectiveness; typical cost $25,000–$50,000. Succession planning tools handling employee assessment data are expected to hold SOC 2 Type II certification.

**SHRM Succession Planning Framework**
- URL: https://www.shrm.org/topics-tools/tools/toolkits/modernize-succession-planning
- The Society for Human Resource Management's practitioner guidelines for succession planning governance, process design, and calibration facilitation. Defines standard readiness levels, 9-box grid usage (used by ~70% of Fortune 500 companies per SHRM benchmarking data), and 12-to-36-month succession preparation cycles. Freely available reference.

---

## Similar Products — Developer Documentation & APIs

### SAP SuccessFactors Succession & Development

- **Description:** The market-leading succession and development module within SAP's HCM suite, covering talent pools, 9-box grid calibration, succession org charts, and AI-based successor matching (SAP Business AI, 1H 2026).
- **API Documentation:** https://api.sap.com/package/SuccessFactorsSuccessionDevelopment/odata
- **API Reference Guide (OData V2):** https://help.sap.com/docs/SAP_SUCCESSFACTORS_PLATFORM/d599f15995d348a1b45ba5603e2aba9b/03e1fc3791684367a6a76a614a2916de.html
- **API Reference Guide (OData V4):** https://help.sap.com/docs/SAP_SUCCESSFACTORS_PLATFORM/9f5f060351034d98990213d077dab38a/b96a4731870e48d090bfbe47b045d259.html
- **Standards:** OData V2 and V4 (OASIS standard); HR Open Standards for external integrations; REST/JSON for newer API surfaces
- **Authentication:** OAuth 2.0 (client credentials and authorisation code flows)
- **Notes:** Nomination history for talent pools is accessible via the OData API and People Analytics Stories. The succession-specific OData entities are the primary integration surface for reading pool membership and readiness ratings.

### Workday HCM — Succession Planning

- **Description:** Succession planning embedded within Workday HCM; uses Workday's Skills Cloud ML model for successor identification and includes real-time bench depth analytics.
- **API Documentation:** https://community.workday.com/sites/default/files/file-hosting/productionapi/index.html (Workday Web Services Directory v46.1)
- **Integration Guide:** https://www.getknit.dev/blog/workday-api-integration-in-depth
- **SDKs/Libraries:** No official SDK; third-party connectors via Workday Extend or integration middleware (Boomi, MuleSoft, Workato)
- **Standards:** REST/JSON (newer APIs); SOAP/XML (legacy Workday Web Services); OAuth 2.0 for all REST endpoints
- **Authentication:** OAuth 2.0 (access tokens derived from OAuth refresh tokens); tokens are scoped per registered API client in the Workday portal
- **Notes:** Workday REST APIs use OAuth 2.0; SOAP WWS APIs use basic auth or OAuth. For succession-specific data, the Talent Management API surface covers successor nominations, talent pools, and readiness ratings.

### BambooHR API

- **Description:** A widely adopted HRIS for SMB and mid-market companies (200–2,000 employees); primary integration target for the AI-native succession tool given the target segment alignment.
- **API Documentation:** https://documentation.bamboohr.com/reference
- **Getting Started / Auth:** https://documentation.bamboohr.com/docs/getting-started
- **Technical Overview:** https://documentation.bamboohr.com/docs/api-details
- **Standards:** REST/JSON; no GraphQL or OData surface
- **Authentication:** HTTP Basic Auth using an API key as the username (API key generated from the user profile in BambooHR); approximately 100 requests/minute per key
- **Notes:** BambooHR API provides employee demographics, job information, performance data, and custom fields. Succession-specific data (if any) is managed as custom fields. Integration point for employee data import into the succession planning tool.

### ADP Workforce Now API

- **Description:** ADP's HRIS and payroll platform covering mid-market to enterprise; strong payroll and benefits integration surface alongside employee demographic and talent data.
- **API Documentation:** https://developers.adp.com/
- **API Catalog (Workforce Now):** https://developers.adp.com/articles/guides/adp-workforce-now-api-catalog
- **Talent Profile API Guide:** https://developers.adp.com/articles/guides/talent-profile-licenses-api-guide-for-adp-workforce-now
- **Standards:** REST/JSON; OAuth 2.0 bearer tokens; OpenAPI specification for newer endpoints
- **Authentication:** OAuth 2.0 (client credentials flow for server-to-server integration); mutual TLS (mTLS) may be required for production connections
- **Notes:** ADP provides employee demographics, job history, and talent profile data. TalentGuard is already integrated with ADP Marketplace, making ADP an important HRIS connector for the succession tool.

### Eightfold AI — Talent Intelligence Platform

- **Description:** AI-native talent intelligence platform with the most sophisticated skills-based successor identification in the market; scans the full workforce without requiring pre-nominated candidates.
- **API Documentation:** https://apidocs.eightfold.ai/
- **Standards:** REST/JSON; no public OpenAPI spec confirmed
- **Authentication:** API key (specific OAuth implementation not confirmed in public documentation)
- **Notes:** Eightfold's API is primarily designed for enterprise HRIS integration partners (Workday, SAP SF, Oracle HCM); not a freely accessible public API. Relevant as a reference architecture for AI-based talent intelligence API design rather than a direct integration target. Eightfold holds patents on skills inference methodology — open-source implementations must use independent approaches (ESCO/O*NET taxonomies).

### TalentGuard

- **Description:** Purpose-built succession planning and career pathing platform targeting mid-market and enterprise; available in the ADP Marketplace and offers REST API access.
- **API/Integration Overview:** https://www.talentguard.com/platform/integrations
- **Standards:** REST/JSON for API integrations; flat-file/SFTP for scheduled sync
- **Authentication:** API key (specific OAuth details not publicly documented)
- **Notes:** TalentGuard supports real-time API sync and SFTP-based scheduled syncs. No public developer portal with full API reference confirmed; enterprise clients access API documentation via direct engagement. Relevant as a product architecture reference.

### Cornerstone OnDemand

- **Description:** Talent management suite with succession, learning, and performance modules; recently rebranded as Cornerstone Galaxy with unified AI-powered talent intelligence.
- **API Overview:** https://www.cornerstoneondemand.com/platform/extend-integrations/
- **API Documentation:** https://apitracker.io/a/cornerstoneondemand
- **Standards:** REST/JSON; proprietary API surface; no confirmed public OpenAPI spec
- **Authentication:** OAuth 2.0 (client credentials); API key authentication for legacy endpoints
- **Notes:** Cornerstone's API covers learning content, performance ratings, and succession pool data. Developer documentation requires partner registration. Integration surface is relevant for development plan execution linking from succession gaps.

### Finch — Unified Employment API

- **Description:** A unified API aggregator that normalises data from 250+ HRIS, payroll, and ATS systems into a single REST API. Significantly reduces the integration burden for building multi-HRIS succession tools.
- **API Documentation:** https://developer.tryfinch.com/
- **Integration Guide:** https://www.tryfinch.com/finch-api
- **SDKs/Libraries:** JavaScript, Python, Java, Kotlin, Go (official backend SDKs)
- **Standards:** REST/JSON; normalised unified data model across all HRIS providers; OpenAPI spec available
- **Authentication:** OAuth 2.0 (Finch handles provider-level auth; integrators use Finch's unified OAuth flow)
- **Notes:** Finch is the most practical path to supporting BambooHR, ADP, Workday, and 200+ other HRIS systems through a single integration. Covers employee demographics, employment status, and job data — the core data needed for succession candidate identification. Used by TalentGuard and others for HRIS connectivity.

---

## Notes

**ESCO vs. O\*NET for Skills Taxonomy:** EU-focused deployments should use the ESCO API (multilingual, 28 languages, free under EC open data licence). US-focused deployments may prefer O*NET (US Department of Labor, 900+ occupations, free). An open-source succession tool should support both via a pluggable taxonomy layer, as noted in the features.md recommended scope.

**Emerging Standard — ISO/TS 30433 Adoption Curve:** ISO/TS 30433:2021 is gaining traction as investors and regulators increasingly require standardised human capital disclosures. The 2025 upgrade of ISO 30414 to mandatory requirements is expected to drive adoption significantly in 2026–2027. Building ISO/TS 30433 metric export into the tool from the outset (rather than retrofitting) is strongly recommended.

**HR Open Standards JSON Transition:** The HR Open Standards Consortium is actively transitioning from XML-first to JSON-first specifications. As of 2024–2025, JSON schemas are the preferred format for new implementations; legacy HR-XML implementations remain valid for existing integrations but should not be used for new development.

**GDPR Article 22 and AI Succession Decisions:** Any AI-generated successor recommendation that could influence employment outcomes (promotion, development investment, remuneration) must include a human review step with authority to override the AI recommendation. This is a legal requirement in the EU, not an optional safeguard. The tool's UX must surface this requirement explicitly — for example, by preventing automatic actions based solely on AI readiness scores without documented human confirmation.
