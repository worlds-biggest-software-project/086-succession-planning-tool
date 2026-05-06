# Succession Planning Tool

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An AI-native, open-source succession planning platform that identifies high-potential employees and maps career paths without enterprise HCM lock-in.

The Succession Planning Tool helps HR leaders, HR business partners, and executives identify key positions, build successor benches, and assess role readiness across the workforce. It targets the underserved mid-market segment (200–2,000 employees) where enterprise HCM modules are over-engineered and overpriced, and SMB tools lack predictive depth. The entire existing succession planning market is proprietary SaaS — this project fills a genuine open-source white space.

---

## Why Succession Planning Tool?

- Enterprise HCM succession modules (SAP SuccessFactors, Oracle HCM, Workday) cost $8–$30 PEPM and require $100K–$2M+ implementations taking 6–18 months — out of reach for mid-market firms.
- AI talent intelligence platforms like Eightfold AI charge $150K–$500K+/year and degrade in accuracy below 500 employees, leaving smaller organisations without credible AI-driven options.
- SMB tools (Trakstar, emPerform) offer simple UX but ship shallow analytics and no predictive capability — managers get a 9-box grid and little else.
- No open-source succession planning tool exists at any capability tier; the entire market is proprietary, with no self-hosted option for data-sovereign organisations.
- Existing tools rely on manager-entered ratings that are biased and point-in-time, and only surface pre-nominated candidates — hidden high-potentials across the workforce stay invisible.

---

## Key Features

### Core Succession Planning

- Position-based succession plans with designated key positions, ranked successors, and readiness tiers (Ready Now, 1–2 Years, 3–5 Years).
- Configurable 9-box grid (performance-versus-potential matrix) with collaborative annotation for calibration sessions.
- Talent pools by role family, leadership tier, business unit, or custom criteria with candidate management and notes.
- Bench strength dashboard with position-level coverage heat map and single-point succession risk alerts.
- Role-based access controls for CHRO, HRBP, manager, and executive views.

### AI-Driven Successor Identification

- Full-workforce successor scanning using skills adjacency matching against open ESCO and O*NET taxonomies, independent of any proprietary model.
- Continuous readiness scoring driven by live HRIS signals (performance data, project participation, learning completions) rather than annual rating cycles.
- Natural language succession queries (for example, "Who covers the CFO role if Sarah leaves?").
- Bench strength scenario modeling: simulate simultaneous departure scenarios and compute resulting coverage impact.

### Equity, Risk & Reporting

- Configurable diversity and equity lens on succession pools with representation gap reporting across the pipeline.
- Risk concentration dashboard highlighting positions with single successors or no bench coverage.
- Auto-generated board-ready succession risk reports with heat map visualisations, exportable to PDF and PowerPoint.
- GDPR Article 22-compliant workflows that keep human review as a documented step in any AI-driven decision.

### Integrations & Deployment

- REST API with initial connectors for Workday, BambooHR, and ADP for employee and performance data import.
- HR Open Standards (HR-XML) data interchange support for portability.
- Self-hosted deployment option for data-sovereign organisations alongside a managed cloud option.

---

## AI-Native Advantage

Unlike incumbents that require manager-nominated candidates and annual calibration cycles, this tool scans the full employee base for hidden high-potentials using skills adjacency graphs built on open taxonomies (ESCO, O*NET) — sidestepping Eightfold's patented methods. Readiness becomes a continuously updated signal drawn from performance, learning, and project data rather than a once-a-year rating exercise. AI also enables bench-strength scenario modeling and natural language succession queries — capabilities that today's mid-market tools only deliver via expensive professional services, if at all.

---

## Tech Stack & Deployment

- Deployment modes: self-hosted (for data-sovereign organisations) and managed cloud.
- Open standards: HR Open Standards (HR-XML) for data exchange; ESCO and O*NET for skills taxonomies; SHRM succession planning framework as a reference; ISO 30414 metrics export for human capital disclosure.
- Integration approach: REST API with pluggable HRIS connectors (Workday, BambooHR, ADP initial targets); LMS integration for development plan execution.
- Compliance posture: GDPR / CCPA aligned with documented lawful basis, Article 22 human-review steps, and a SOC 2 Type II target for cloud deployments.

---

## Market Context

The global succession planning software market was $702M in 2024 and is projected to reach $1B by 2029 at ~7.2% CAGR (Apps Run the World), with broader talent-management-inclusive estimates ranging from $2.46B in 2024 to $4.25B by 2032 (Business Research Insights) and up to $8.6B by 2034 (DataIntelo). SAP leads with ~12.6% share, followed by Oracle, Workday, and UKG; Eightfold AI raised $220M Series E at a $2.1B valuation. Primary buyers are CHROs and Chief People Officers (succession strategy, board reporting), HR Business Partners (calibration and pool management), and L&D Directors (linking gaps to development).

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
