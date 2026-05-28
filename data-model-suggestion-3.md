# Data Model Suggestion 3: Hybrid Relational + JSONB

> Project: Succession Planning Tool · Created: 2026-05-12

## Philosophy

This model uses a relational backbone for core entities and relationships (positions, employees, succession plans, nominations) but delegates variable, jurisdiction-specific, and rapidly evolving fields to JSONB columns. The core schema is lean — fewer tables than a fully normalized model — while JSONB columns absorb the complexity that would otherwise require dozens of lookup tables or frequent migrations.

This pattern is widely used in modern SaaS platforms that operate across multiple jurisdictions, industries, or customer configurations. In the succession planning domain, it addresses a key challenge: different organisations define readiness differently, use different diversity categories (which vary by jurisdiction under GDPR vs. CCPA vs. Australian Fair Work Act), track different competency frameworks, and configure their 9-box grids with different labels and scales. Rather than pre-defining every possible permutation as columns, JSONB fields allow each deployment to extend the schema without migrations.

PostgreSQL's JSONB support — including GIN indexes, containment operators, and jsonpath queries — makes this approach performant for the query patterns succession planning requires: filtering talent pools by custom attributes, searching for employees with specific skill combinations, and generating configurable reports.

**Best for:** Multi-tenant SaaS deployments serving organisations with different succession planning configurations, readiness frameworks, and jurisdiction-specific diversity reporting requirements.

**Trade-offs:**
- Pro: Fewer tables (~20) and simpler JOINs than fully normalized model
- Pro: New fields can be added without schema migrations — just extend the JSONB
- Pro: Multi-jurisdiction diversity categories, custom readiness tiers, and configurable 9-box scales handled without code changes
- Pro: Faster MVP development — core schema stabilises quickly while JSONB absorbs iteration
- Pro: Natural fit for API responses — JSONB fields serialize directly to JSON
- Con: JSONB fields lack referential integrity — foreign key relationships inside JSON are not enforced by the database
- Con: Requires application-level validation for JSONB structure (JSON Schema validation)
- Con: Complex JSONB queries can be slower than equivalent relational JOINs without careful indexing
- Con: JSONB data is harder to analyse with standard SQL reporting tools
- Con: Risk of JSONB columns becoming dumping grounds — requires disciplined field governance

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ISO/TS 30433:2021 | Succession metrics computed from relational fields; JSONB `custom_metrics` column allows tenant-specific supplementary metrics |
| ISO 30414:2025 | Core ISO metrics are relational columns; jurisdiction-specific disclosure requirements stored in JSONB configuration |
| ISO 30400:2022 | Relational column names follow ISO 30400 vocabulary; JSONB keys use the same terms |
| ESCO Classification | Employee skills reference ESCO URIs in the relational `skill` table; JSONB `skill_details` holds proficiency metadata |
| O*NET | Positions can reference O*NET codes; occupation-specific attributes stored in JSONB |
| HR Open Standards | JSONB payloads are designed to serialize directly to HR-JSON assessment and talent schemas |
| GDPR / CCPA | Diversity dimensions stored in JSONB with jurisdiction-aware category sets; tenant configuration controls which dimensions are collected |
| SHRM Framework | Default readiness tier configuration follows SHRM; tenants can override via JSONB configuration |

---

## Tenant Configuration

```sql
-- Tenant-level configuration (controls JSONB field structures)
CREATE TABLE tenant (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    slug VARCHAR(100) NOT NULL UNIQUE,
    -- Configurable succession framework
    config JSONB NOT NULL DEFAULT '{}',
    -- config example:
    -- {
    --   "readiness_tiers": ["ready_now", "6_months", "1_to_2_years", "3_to_5_years"],
    --   "nine_box_scale": { "performance_labels": ["Below", "Meets", "Exceeds"], "potential_labels": ["Limited", "Growth", "High"] },
    --   "diversity_dimensions": ["gender", "ethnicity", "disability_status"],
    --   "diversity_categories": {
    --     "gender": ["male", "female", "non_binary", "prefer_not_to_say"],
    --     "ethnicity": ["...jurisdiction-specific categories..."]
    --   },
    --   "skills_taxonomy": "esco",
    --   "custom_readiness_fields": ["leadership_competency", "technical_depth", "stakeholder_management"],
    --   "iso_30414_enabled": true
    -- }
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Core Entities

```sql
CREATE TABLE organisation_unit (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenant(id),
    parent_id UUID REFERENCES organisation_unit(id),
    name VARCHAR(255) NOT NULL,
    unit_type VARCHAR(50) NOT NULL,
    country_code CHAR(2), -- ISO 3166-1
    metadata JSONB NOT NULL DEFAULT '{}', -- tenant-specific fields (cost centre, region code, etc.)
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_org_unit_tenant ON organisation_unit(tenant_id);
CREATE INDEX idx_org_unit_parent ON organisation_unit(parent_id);

CREATE TABLE position (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenant(id),
    organisation_unit_id UUID NOT NULL REFERENCES organisation_unit(id),
    title VARCHAR(255) NOT NULL,
    position_code VARCHAR(50),
    level VARCHAR(50),
    is_key_position BOOLEAN NOT NULL DEFAULT false,
    esco_occupation_uri VARCHAR(500),
    onet_code VARCHAR(20),
    -- JSONB for position-specific attributes that vary by tenant
    attributes JSONB NOT NULL DEFAULT '{}',
    -- attributes example:
    -- {
    --   "job_family": "Finance",
    --   "criticality": "mission_critical",
    --   "replacement_urgency": "immediate",
    --   "required_certifications": ["CPA", "CFA"],
    --   "succession_priority": 1
    -- }
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, position_code)
);

CREATE INDEX idx_position_tenant ON position(tenant_id);
CREATE INDEX idx_position_key ON position(is_key_position) WHERE is_key_position = true;
CREATE INDEX idx_position_attrs ON position USING GIN (attributes);

CREATE TABLE employee (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenant(id),
    external_hris_id VARCHAR(100),
    hris_source VARCHAR(50),
    first_name VARCHAR(100) NOT NULL,
    last_name VARCHAR(100) NOT NULL,
    email VARCHAR(255),
    current_position_id UUID REFERENCES position(id),
    organisation_unit_id UUID REFERENCES organisation_unit(id),
    hire_date DATE,
    employment_status VARCHAR(20) NOT NULL DEFAULT 'active',
    -- JSONB for employee attributes that vary by tenant / jurisdiction
    profile JSONB NOT NULL DEFAULT '{}',
    -- profile example:
    -- {
    --   "certifications": [{"name": "PMP", "expires": "2027-06-01"}],
    --   "languages": [{"code": "en", "proficiency": "native"}, {"code": "de", "proficiency": "B2"}],
    --   "flight_risk_score": 0.35,
    --   "years_in_current_role": 2.5,
    --   "custom_fields": { "leadership_program_cohort": "2025-Q3" }
    -- }
    -- Demographics stored in JSONB with jurisdiction-aware categories
    demographics JSONB NOT NULL DEFAULT '{}',
    -- demographics example (GDPR-compliant, optional self-reported):
    -- {
    --   "gender": "female",
    --   "ethnicity": "asian",
    --   "age_band": "35-44",
    --   "disability_status": "none_declared",
    --   "consent_given_at": "2026-01-15T10:30:00Z"
    -- }
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_employee_tenant ON employee(tenant_id);
CREATE INDEX idx_employee_position ON employee(current_position_id);
CREATE INDEX idx_employee_profile ON employee USING GIN (profile);
CREATE INDEX idx_employee_hris ON employee(tenant_id, hris_source, external_hris_id);
```

## Skills

```sql
CREATE TABLE skill (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID REFERENCES tenant(id), -- NULL = global taxonomy skill
    name VARCHAR(255) NOT NULL,
    taxonomy_source VARCHAR(20) NOT NULL,
    external_uri VARCHAR(500),
    skill_type VARCHAR(30) NOT NULL,
    metadata JSONB NOT NULL DEFAULT '{}', -- reusability_level, related skills, etc.
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_skill_taxonomy ON skill(taxonomy_source, external_uri);

-- Employee skills with flexible assessment metadata
CREATE TABLE employee_skill (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    employee_id UUID NOT NULL REFERENCES employee(id) ON DELETE CASCADE,
    skill_id UUID NOT NULL REFERENCES skill(id),
    proficiency_level INTEGER CHECK (proficiency_level BETWEEN 1 AND 5),
    source VARCHAR(30) NOT NULL,
    -- JSONB for assessment details that vary by source
    assessment_details JSONB NOT NULL DEFAULT '{}',
    -- assessment_details examples:
    -- For AI-inferred: { "confidence": 0.82, "algorithm": "v2.1", "evidence": ["project-xyz", "learning-abc"] }
    -- For certification: { "certification_name": "AWS SA", "issue_date": "2025-09-01", "expiry": "2028-09-01" }
    -- For manager-assessed: { "assessed_by": "uuid", "assessment_date": "2026-03-15", "notes": "..." }
    assessed_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (employee_id, skill_id, source)
);

CREATE INDEX idx_emp_skill_employee ON employee_skill(employee_id);
CREATE INDEX idx_emp_skill_details ON employee_skill USING GIN (assessment_details);
```

## Succession Plans & Nominations

```sql
CREATE TABLE succession_plan (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenant(id),
    position_id UUID NOT NULL REFERENCES position(id),
    owner_id UUID NOT NULL REFERENCES employee(id),
    status VARCHAR(20) NOT NULL DEFAULT 'draft',
    review_frequency_months INTEGER NOT NULL DEFAULT 12,
    last_reviewed_at TIMESTAMPTZ,
    -- JSONB for plan-specific configuration
    plan_config JSONB NOT NULL DEFAULT '{}',
    -- plan_config example:
    -- {
    --   "minimum_bench_depth": 3,
    --   "require_diversity_in_bench": true,
    --   "auto_alert_on_single_successor": true,
    --   "custom_readiness_criteria": {
    --     "leadership_assessment": "required",
    --     "cross_functional_experience": "preferred"
    --   }
    -- }
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_plan_tenant ON succession_plan(tenant_id);
CREATE INDEX idx_plan_position ON succession_plan(position_id);

CREATE TABLE successor_nomination (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    succession_plan_id UUID NOT NULL REFERENCES succession_plan(id) ON DELETE CASCADE,
    employee_id UUID NOT NULL REFERENCES employee(id),
    nominated_by UUID NOT NULL REFERENCES employee(id),
    readiness_tier VARCHAR(30) NOT NULL, -- not CHECK-constrained; tier labels from tenant config
    rank_order INTEGER,
    nomination_status VARCHAR(20) NOT NULL DEFAULT 'nominated',
    -- JSONB for flexible readiness assessment data
    readiness_assessment JSONB NOT NULL DEFAULT '{}',
    -- readiness_assessment example:
    -- {
    --   "risk_of_loss": "medium",
    --   "impact_of_loss": "high",
    --   "skills_gap_count": 2,
    --   "development_actions_completed": 4,
    --   "development_actions_total": 7,
    --   "custom_scores": {
    --     "leadership_competency": 4,
    --     "technical_depth": 3,
    --     "stakeholder_management": 5
    --   },
    --   "manager_comments": "Strong candidate, needs cross-functional exposure",
    --   "last_assessment_date": "2026-03-15"
    -- }
    nominated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (succession_plan_id, employee_id)
);

CREATE INDEX idx_nomination_plan ON successor_nomination(succession_plan_id);
CREATE INDEX idx_nomination_employee ON successor_nomination(employee_id);
CREATE INDEX idx_nomination_readiness ON successor_nomination USING GIN (readiness_assessment);
```

## 9-Box & Calibration

```sql
CREATE TABLE assessment (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenant(id),
    employee_id UUID NOT NULL REFERENCES employee(id),
    assessment_period_start DATE NOT NULL,
    assessment_period_end DATE NOT NULL,
    performance_rating INTEGER NOT NULL CHECK (performance_rating BETWEEN 1 AND 5), -- wider range; actual scale from tenant config
    potential_rating INTEGER NOT NULL CHECK (potential_rating BETWEEN 1 AND 5),
    source VARCHAR(30) NOT NULL,
    assessed_by UUID REFERENCES employee(id),
    -- JSONB for tenant-configurable assessment dimensions
    dimensions JSONB NOT NULL DEFAULT '{}',
    -- dimensions example:
    -- {
    --   "leadership": 4,
    --   "innovation": 3,
    --   "collaboration": 5,
    --   "results_orientation": 4,
    --   "manager_narrative": "Consistently exceeds expectations...",
    --   "nine_box_override": null,
    --   "calibration_notes": null
    -- }
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_assessment_tenant ON assessment(tenant_id);
CREATE INDEX idx_assessment_employee ON assessment(employee_id);
CREATE INDEX idx_assessment_period ON assessment(assessment_period_end DESC);
CREATE INDEX idx_assessment_dims ON assessment USING GIN (dimensions);

CREATE TABLE calibration_session (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenant(id),
    name VARCHAR(255) NOT NULL,
    facilitator_id UUID NOT NULL REFERENCES employee(id),
    organisation_unit_id UUID REFERENCES organisation_unit(id),
    session_date DATE NOT NULL,
    status VARCHAR(20) NOT NULL DEFAULT 'planned',
    -- JSONB for session outcomes and configuration
    session_data JSONB NOT NULL DEFAULT '{}',
    -- session_data example:
    -- {
    --   "participants": [{"employee_id": "uuid", "role": "reviewer"}],
    --   "decisions": [
    --     {
    --       "employee_id": "uuid",
    --       "original_performance": 2, "adjusted_performance": 3,
    --       "original_potential": 2, "adjusted_potential": 2,
    --       "rationale": "Consistent delivery on stretch projects",
    --       "consensus_reached": true
    --     }
    --   ],
    --   "total_employees_reviewed": 25,
    --   "adjustments_made": 8
    -- }
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_calibration_tenant ON calibration_session(tenant_id);
```

## Talent Pools

```sql
CREATE TABLE talent_pool (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenant(id),
    name VARCHAR(255) NOT NULL,
    pool_type VARCHAR(30) NOT NULL,
    owner_id UUID NOT NULL REFERENCES employee(id),
    is_active BOOLEAN NOT NULL DEFAULT true,
    -- JSONB for pool criteria and configuration
    criteria JSONB NOT NULL DEFAULT '{}',
    -- criteria example:
    -- {
    --   "auto_populate": true,
    --   "filters": {
    --     "nine_box_positions": [6, 8, 9],
    --     "min_tenure_years": 2,
    --     "required_skills": ["uuid1", "uuid2"],
    --     "organisation_units": ["uuid3"]
    --   },
    --   "capacity": 20,
    --   "description": "Future VPs - high potential leaders"
    -- }
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_pool_tenant ON talent_pool(tenant_id);
CREATE INDEX idx_pool_criteria ON talent_pool USING GIN (criteria);

CREATE TABLE talent_pool_member (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    talent_pool_id UUID NOT NULL REFERENCES talent_pool(id) ON DELETE CASCADE,
    employee_id UUID NOT NULL REFERENCES employee(id),
    added_by UUID NOT NULL REFERENCES employee(id),
    readiness_tier VARCHAR(30),
    member_data JSONB NOT NULL DEFAULT '{}',
    -- member_data example:
    -- { "notes": "Strong analytical skills", "development_focus": "people management" }
    added_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (talent_pool_id, employee_id)
);
```

## AI Recommendations

```sql
CREATE TABLE ai_recommendation (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenant(id),
    position_id UUID NOT NULL REFERENCES position(id),
    employee_id UUID NOT NULL REFERENCES employee(id),
    algorithm_version VARCHAR(50) NOT NULL,
    overall_fit_score NUMERIC(5,4),
    -- JSONB for detailed scoring breakdown
    scoring JSONB NOT NULL DEFAULT '{}',
    -- scoring example:
    -- {
    --   "skills_match": 0.87,
    --   "readiness": 0.72,
    --   "trajectory": 0.91,
    --   "skill_gaps": [
    --     {"skill_id": "uuid", "skill_name": "Strategic Planning", "current": 2, "required": 4}
    --   ],
    --   "strengths": ["Financial Acumen", "Stakeholder Management"],
    --   "explanation": "Strong skills alignment with 2 gaps in strategic areas..."
    -- }
    -- GDPR Article 22 compliance
    human_review_status VARCHAR(20) NOT NULL DEFAULT 'pending',
    reviewer_id UUID REFERENCES employee(id),
    review_data JSONB NOT NULL DEFAULT '{}',
    -- review_data example:
    -- { "decision": "approved", "notes": "Confirmed after HRBP discussion", "reviewed_at": "2026-04-01T14:30:00Z" }
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_ai_rec_tenant ON ai_recommendation(tenant_id);
CREATE INDEX idx_ai_rec_position ON ai_recommendation(position_id);
CREATE INDEX idx_ai_rec_review ON ai_recommendation(human_review_status);
CREATE INDEX idx_ai_rec_scoring ON ai_recommendation USING GIN (scoring);
```

## Reporting & Audit

```sql
-- ISO 30433 metrics (relational for consistent querying)
CREATE TABLE succession_metrics (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenant(id),
    reporting_period_start DATE NOT NULL,
    reporting_period_end DATE NOT NULL,
    organisation_unit_id UUID REFERENCES organisation_unit(id),
    succession_effectiveness_rate NUMERIC(5,4),
    successor_coverage_rate NUMERIC(5,4),
    succession_depth_ready_now NUMERIC(5,2),
    succession_depth_1_to_3_years NUMERIC(5,2),
    succession_depth_4_to_5_years NUMERIC(5,2),
    -- JSONB for tenant-specific supplementary metrics
    custom_metrics JSONB NOT NULL DEFAULT '{}',
    -- custom_metrics example:
    -- { "diversity_coverage": 0.65, "avg_time_in_pipeline_months": 18, "internal_fill_rate": 0.72 }
    computed_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_metrics_tenant ON succession_metrics(tenant_id);
CREATE INDEX idx_metrics_period ON succession_metrics(reporting_period_end DESC);

-- Audit log
CREATE TABLE audit_log (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenant(id),
    user_id UUID,
    action VARCHAR(50) NOT NULL,
    entity_type VARCHAR(50) NOT NULL,
    entity_id UUID NOT NULL,
    changes JSONB NOT NULL DEFAULT '{}', -- { "old": {...}, "new": {...} }
    request_metadata JSONB NOT NULL DEFAULT '{}', -- { "ip": "...", "user_agent": "..." }
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_tenant ON audit_log(tenant_id);
CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
CREATE INDEX idx_audit_created ON audit_log(created_at DESC);

-- HRIS integration
CREATE TABLE hris_connection (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenant(id),
    provider VARCHAR(30) NOT NULL,
    connection_name VARCHAR(255) NOT NULL,
    config JSONB NOT NULL DEFAULT '{}', -- auth type, sync frequency, field mappings
    -- config example:
    -- {
    --   "auth_type": "oauth2",
    --   "sync_frequency_minutes": 1440,
    --   "field_mappings": {
    --     "employee_id": "workerId",
    --     "first_name": "legalName.givenName",
    --     "performance_rating": "talentProfile.performanceRating"
    --   }
    -- }
    credentials_encrypted BYTEA,
    last_sync_at TIMESTAMPTZ,
    last_sync_status VARCHAR(20),
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Access control
CREATE TABLE user_account (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenant(id),
    employee_id UUID REFERENCES employee(id),
    email VARCHAR(255) NOT NULL,
    auth_provider VARCHAR(30) NOT NULL DEFAULT 'oidc',
    external_auth_id VARCHAR(255),
    roles JSONB NOT NULL DEFAULT '[]',
    -- roles example:
    -- [
    --   {"role": "hrbp", "scope": {"organisation_unit_id": "uuid"}},
    --   {"role": "chro", "scope": null}
    -- ]
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, email)
);

CREATE INDEX idx_user_tenant ON user_account(tenant_id);
CREATE INDEX idx_user_roles ON user_account USING GIN (roles);
```

## Example Queries

### Find employees with specific skill combinations (JSONB containment)

```sql
-- Find employees with leadership skills who are in the top-right 9-box quadrant
SELECT e.id, e.first_name, e.last_name, a.performance_rating, a.potential_rating
FROM employee e
JOIN assessment a ON a.employee_id = e.id
WHERE e.tenant_id = '<tenant-uuid>'
  AND a.performance_rating >= 2
  AND a.potential_rating >= 2
  AND a.assessment_period_end = (
      SELECT MAX(assessment_period_end) FROM assessment WHERE employee_id = e.id
  )
  AND EXISTS (
      SELECT 1 FROM employee_skill es
      JOIN skill s ON s.id = es.skill_id
      WHERE es.employee_id = e.id
        AND s.name ILIKE '%leadership%'
        AND es.proficiency_level >= 3
  )
ORDER BY a.potential_rating DESC, a.performance_rating DESC;
```

### Tenant-configurable diversity reporting

```sql
-- Generate diversity breakdown for a talent pool using tenant-configured dimensions
SELECT
    tp.name AS pool_name,
    e.demographics->>'gender' AS gender,
    COUNT(*) AS member_count,
    COUNT(*) FILTER (WHERE sn.readiness_tier = 'ready_now') AS ready_now
FROM talent_pool_member tpm
JOIN talent_pool tp ON tp.id = tpm.talent_pool_id
JOIN employee e ON e.id = tpm.employee_id
LEFT JOIN successor_nomination sn ON sn.employee_id = e.id
WHERE tp.tenant_id = '<tenant-uuid>'
  AND tp.id = '<pool-uuid>'
  AND e.demographics ? 'gender' -- only include employees who have declared
GROUP BY tp.name, e.demographics->>'gender';
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Configuration | 1 | tenant (with JSONB config) |
| Core Entities | 3 | organisation_unit, position, employee (with JSONB attributes) |
| Skills | 2 | skill, employee_skill (with JSONB assessment_details) |
| Succession | 2 | succession_plan, successor_nomination (with JSONB readiness) |
| Calibration | 2 | assessment, calibration_session (with JSONB dimensions/data) |
| Talent Pools | 2 | talent_pool, talent_pool_member |
| AI | 1 | ai_recommendation (with JSONB scoring) |
| Reporting & Audit | 2 | succession_metrics, audit_log |
| Integration & Access | 2 | hris_connection, user_account (with JSONB roles) |
| **Total** | **17** | Significantly fewer tables than normalized; flexibility via JSONB |

---

## Key Design Decisions

1. **Tenant-level configuration drives JSONB structure** — the `tenant.config` JSONB column defines what readiness tiers, diversity dimensions, 9-box scales, and custom fields each tenant uses. Application code validates JSONB payloads against the tenant's configuration.

2. **Demographics in JSONB, not separate table** — employee demographics are stored as a JSONB column on the employee table rather than a separate table. This simplifies queries and allows jurisdiction-specific category sets (GDPR-region tenants track different categories than US tenants). Access control is enforced at the application layer using the `roles` JSONB on `user_account`.

3. **Readiness tier is VARCHAR, not CHECK-constrained** — unlike the normalized model, readiness tiers are not enforced at the database level because different tenants define different tiers. Validation happens at the application layer against `tenant.config.readiness_tiers`.

4. **Calibration decisions inside session JSONB** — rather than separate `calibration_participant` and `calibration_decision` tables, calibration data is stored as a JSONB array inside `calibration_session.session_data`. This keeps all session data together and avoids 3 tables for a single conceptual operation.

5. **GIN indexes on all JSONB columns** — every JSONB column used in queries has a GIN index, enabling efficient containment (`@>`) and existence (`?`) queries without full-table scans.

6. **Roles as JSONB array** — rather than separate `app_role` and `user_role` tables, roles are stored as a JSONB array on `user_account` with optional org-unit scoping. This reduces table count and simplifies role checking at the API layer.

7. **HRIS field mappings in JSONB** — the `hris_connection.config` JSONB includes field mappings between the HRIS provider's field names and the tool's schema. This allows each tenant to map their BambooHR custom fields, Workday attributes, or ADP talent profile data without code changes.

8. **ISO 30433 metrics are relational** — despite the JSONB-heavy approach, ISO metrics are stored as typed numeric columns for reliable aggregation and reporting. A `custom_metrics` JSONB column handles tenant-specific supplementary metrics.

9. **17 tables vs. 27 in the normalized model** — the JSONB approach consolidates approximately 10 tables worth of lookup and junction data into JSONB columns, resulting in a simpler schema that is easier to deploy, migrate, and operate.
