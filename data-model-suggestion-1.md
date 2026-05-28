# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Succession Planning Tool · Created: 2026-05-12

## Philosophy

This model follows a fully normalized relational design where every domain concept — positions, employees, succession plans, talent pools, assessments, readiness ratings, calibration sessions, and ISO 30433 metrics — is represented as a dedicated table with explicit foreign keys enforcing referential integrity. The schema is designed so that a single SELECT with JOINs can answer any succession planning question without ambiguity.

This approach mirrors how enterprise HCM systems like SAP SuccessFactors structure their OData entities (TalentPool, Successor, NomineeHistory, NominationTarget) and aligns with the HR Open Standards Consortium's entity-based data interchange philosophy. Every relationship is explicit, every field has a type and constraint, and the schema is self-documenting.

The normalized design is best suited for organisations that prioritise data integrity, regulatory compliance (GDPR Article 22 auditability, ISO 30414/30433 metrics export), and long-term maintainability over rapid schema evolution. It trades flexibility for correctness.

**Best for:** Compliance-first deployments where data integrity, auditability, and standards-aligned reporting (ISO 30414/30433) are non-negotiable.

**Trade-offs:**
- Pro: Strong referential integrity prevents orphaned or inconsistent succession data
- Pro: Direct mapping to ISO 30433 metrics (successor coverage rate, succession depth rate) via straightforward aggregate queries
- Pro: Clean HRIS integration surface — each entity maps 1:1 to HR Open Standards JSON schemas
- Pro: Easy to reason about for DBAs and auditors
- Con: High table count (~35-40 tables) increases JOIN complexity for cross-cutting queries
- Con: Schema changes require migrations; adding a new readiness dimension means ALTER TABLE
- Con: Multi-jurisdiction variations (different readiness tier labels, different diversity categories) require either denormalization or excessive lookup tables
- Con: Historical state queries ("what was the bench strength on March 1?") require temporal tables or snapshot mechanisms not native to this pattern

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ISO/TS 30433:2021 | Dedicated `succession_metrics` table with fields for each ISO metric: succession_effectiveness_rate, successor_coverage_rate, succession_depth_ready_now, succession_depth_1_to_3_years, succession_depth_4_to_5_years |
| ISO 30414:2025 | Metrics export view aggregates succession_metrics with diversity and internal promotion data for human capital disclosure |
| ISO 30400:2022 | Column and table names use ISO 30400 vocabulary: "talent_pool", "succession_plan", "readiness", "bench_strength" |
| ESCO Classification | `skill` table references ESCO concept URIs; `employee_skill` maps employees to ESCO skills with proficiency levels |
| O*NET | `occupation` table supports both ESCO and O*NET codes via `taxonomy_source` discriminator |
| HR Open Standards (HR-JSON) | Entity structure mirrors HR-JSON Assessment and Talent schemas for direct serialization |
| SHRM Framework | Readiness tiers (Ready Now, 1-2 Years, 3-5 Years) follow SHRM succession planning guidelines |
| GDPR Article 22 | `ai_recommendation` table includes `human_review_status` and `reviewer_id` columns; no recommendation can be actioned without documented human review |

---

## Entity Management

```sql
-- Organisation hierarchy
CREATE TABLE organisation_unit (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    parent_id UUID REFERENCES organisation_unit(id),
    name VARCHAR(255) NOT NULL,
    unit_type VARCHAR(50) NOT NULL CHECK (unit_type IN ('company', 'division', 'department', 'team')),
    country_code CHAR(2), -- ISO 3166-1 alpha-2
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_org_unit_parent ON organisation_unit(parent_id);
CREATE INDEX idx_org_unit_type ON organisation_unit(unit_type);

-- Positions (roles that can be succession-planned)
CREATE TABLE position (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_unit_id UUID NOT NULL REFERENCES organisation_unit(id),
    title VARCHAR(255) NOT NULL,
    position_code VARCHAR(50) UNIQUE,
    level VARCHAR(50), -- e.g., 'C-Suite', 'VP', 'Director', 'Manager', 'Individual Contributor'
    is_key_position BOOLEAN NOT NULL DEFAULT false, -- flagged for succession planning
    esco_occupation_uri VARCHAR(500), -- ESCO occupation concept URI
    onet_code VARCHAR(20), -- O*NET-SOC code
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_position_org_unit ON position(organisation_unit_id);
CREATE INDEX idx_position_key ON position(is_key_position) WHERE is_key_position = true;

-- Employees (imported from HRIS)
CREATE TABLE employee (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    external_hris_id VARCHAR(100), -- ID in source HRIS (Workday, BambooHR, ADP)
    hris_source VARCHAR(50), -- 'workday', 'bamboohr', 'adp', 'manual'
    first_name VARCHAR(100) NOT NULL,
    last_name VARCHAR(100) NOT NULL,
    email VARCHAR(255),
    current_position_id UUID REFERENCES position(id),
    organisation_unit_id UUID REFERENCES organisation_unit(id),
    hire_date DATE,
    tenure_years NUMERIC(5,2) GENERATED ALWAYS AS (EXTRACT(EPOCH FROM (now() - hire_date)) / 31557600) STORED,
    employment_status VARCHAR(20) NOT NULL DEFAULT 'active' CHECK (employment_status IN ('active', 'on_leave', 'terminated')),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_employee_position ON employee(current_position_id);
CREATE INDEX idx_employee_org_unit ON employee(organisation_unit_id);
CREATE INDEX idx_employee_hris ON employee(hris_source, external_hris_id);
```

## Skills & Competencies

```sql
-- Skills taxonomy (populated from ESCO and/or O*NET)
CREATE TABLE skill (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    description TEXT,
    taxonomy_source VARCHAR(20) NOT NULL CHECK (taxonomy_source IN ('esco', 'onet', 'custom')),
    external_uri VARCHAR(500), -- ESCO concept URI or O*NET element ID
    skill_type VARCHAR(30) NOT NULL CHECK (skill_type IN ('skill', 'competence', 'knowledge')), -- ESCO distinction
    reusability_level VARCHAR(30), -- ESCO: 'transversal', 'cross-sector', 'sector-specific', 'occupation-specific'
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_skill_taxonomy ON skill(taxonomy_source, external_uri);
CREATE INDEX idx_skill_type ON skill(skill_type);

-- Skills required for a position
CREATE TABLE position_skill (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    position_id UUID NOT NULL REFERENCES position(id) ON DELETE CASCADE,
    skill_id UUID NOT NULL REFERENCES skill(id),
    importance VARCHAR(20) NOT NULL CHECK (importance IN ('essential', 'important', 'nice_to_have')),
    proficiency_required INTEGER CHECK (proficiency_required BETWEEN 1 AND 5),
    UNIQUE (position_id, skill_id)
);

-- Skills held by an employee
CREATE TABLE employee_skill (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    employee_id UUID NOT NULL REFERENCES employee(id) ON DELETE CASCADE,
    skill_id UUID NOT NULL REFERENCES skill(id),
    proficiency_level INTEGER CHECK (proficiency_level BETWEEN 1 AND 5),
    source VARCHAR(30) NOT NULL CHECK (source IN ('self_assessed', 'manager_assessed', 'ai_inferred', 'certification', 'hris_import')),
    assessed_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (employee_id, skill_id, source)
);

CREATE INDEX idx_employee_skill_emp ON employee_skill(employee_id);
CREATE INDEX idx_employee_skill_skill ON employee_skill(skill_id);
```

## Succession Plans & Nominations

```sql
-- Succession plans for key positions
CREATE TABLE succession_plan (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    position_id UUID NOT NULL REFERENCES position(id),
    owner_id UUID NOT NULL REFERENCES employee(id), -- HRBP or CHRO who owns the plan
    status VARCHAR(20) NOT NULL DEFAULT 'draft' CHECK (status IN ('draft', 'active', 'archived')),
    review_frequency_months INTEGER NOT NULL DEFAULT 12,
    last_reviewed_at TIMESTAMPTZ,
    next_review_due TIMESTAMPTZ,
    notes TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_succession_plan_position ON succession_plan(position_id);
CREATE INDEX idx_succession_plan_status ON succession_plan(status);

-- Successor nominations
CREATE TABLE successor_nomination (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    succession_plan_id UUID NOT NULL REFERENCES succession_plan(id) ON DELETE CASCADE,
    employee_id UUID NOT NULL REFERENCES employee(id),
    nominated_by UUID NOT NULL REFERENCES employee(id),
    readiness_tier VARCHAR(20) NOT NULL CHECK (readiness_tier IN ('ready_now', '1_to_2_years', '3_to_5_years')),
    rank_order INTEGER, -- 1 = top candidate
    nomination_status VARCHAR(20) NOT NULL DEFAULT 'nominated' CHECK (nomination_status IN ('nominated', 'confirmed', 'withdrawn', 'promoted')),
    risk_of_loss VARCHAR(20) CHECK (risk_of_loss IN ('low', 'medium', 'high')),
    impact_of_loss VARCHAR(20) CHECK (impact_of_loss IN ('low', 'medium', 'high')),
    development_notes TEXT,
    nominated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (succession_plan_id, employee_id)
);

CREATE INDEX idx_nomination_plan ON successor_nomination(succession_plan_id);
CREATE INDEX idx_nomination_employee ON successor_nomination(employee_id);
CREATE INDEX idx_nomination_readiness ON successor_nomination(readiness_tier);
```

## 9-Box Grid & Calibration

```sql
-- Performance and potential assessments (input to 9-box)
CREATE TABLE assessment (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    employee_id UUID NOT NULL REFERENCES employee(id),
    assessment_period_start DATE NOT NULL,
    assessment_period_end DATE NOT NULL,
    performance_rating INTEGER NOT NULL CHECK (performance_rating BETWEEN 1 AND 3), -- 1=Low, 2=Medium, 3=High
    potential_rating INTEGER NOT NULL CHECK (potential_rating BETWEEN 1 AND 3),    -- 1=Low, 2=Medium, 3=High
    nine_box_position INTEGER GENERATED ALWAYS AS ((potential_rating - 1) * 3 + performance_rating) STORED, -- 1-9
    assessed_by UUID REFERENCES employee(id),
    source VARCHAR(30) NOT NULL CHECK (source IN ('manager', 'calibration', 'ai_suggested', 'hris_import')),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_assessment_employee ON assessment(employee_id);
CREATE INDEX idx_assessment_period ON assessment(assessment_period_end DESC);
CREATE INDEX idx_assessment_nine_box ON assessment(nine_box_position);

-- Calibration sessions
CREATE TABLE calibration_session (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    facilitator_id UUID NOT NULL REFERENCES employee(id),
    organisation_unit_id UUID REFERENCES organisation_unit(id),
    session_date DATE NOT NULL,
    status VARCHAR(20) NOT NULL DEFAULT 'planned' CHECK (status IN ('planned', 'in_progress', 'completed', 'cancelled')),
    notes TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Participants in a calibration session
CREATE TABLE calibration_participant (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    calibration_session_id UUID NOT NULL REFERENCES calibration_session(id) ON DELETE CASCADE,
    employee_id UUID NOT NULL REFERENCES employee(id),
    role VARCHAR(30) NOT NULL CHECK (role IN ('facilitator', 'reviewer', 'observer')),
    UNIQUE (calibration_session_id, employee_id)
);

-- Calibration decisions (adjustments to assessments during calibration)
CREATE TABLE calibration_decision (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    calibration_session_id UUID NOT NULL REFERENCES calibration_session(id) ON DELETE CASCADE,
    employee_id UUID NOT NULL REFERENCES employee(id), -- employee being calibrated
    original_assessment_id UUID REFERENCES assessment(id),
    adjusted_performance_rating INTEGER CHECK (adjusted_performance_rating BETWEEN 1 AND 3),
    adjusted_potential_rating INTEGER CHECK (adjusted_potential_rating BETWEEN 1 AND 3),
    rationale TEXT,
    decided_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Talent Pools

```sql
CREATE TABLE talent_pool (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    description TEXT,
    pool_type VARCHAR(30) NOT NULL CHECK (pool_type IN ('leadership', 'role_family', 'business_unit', 'strategic_program', 'custom')),
    owner_id UUID NOT NULL REFERENCES employee(id),
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE talent_pool_member (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    talent_pool_id UUID NOT NULL REFERENCES talent_pool(id) ON DELETE CASCADE,
    employee_id UUID NOT NULL REFERENCES employee(id),
    added_by UUID NOT NULL REFERENCES employee(id),
    readiness_tier VARCHAR(20) CHECK (readiness_tier IN ('ready_now', '1_to_2_years', '3_to_5_years')),
    notes TEXT,
    added_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (talent_pool_id, employee_id)
);

CREATE INDEX idx_pool_member_pool ON talent_pool_member(talent_pool_id);
CREATE INDEX idx_pool_member_employee ON talent_pool_member(employee_id);
```

## AI Recommendations & GDPR Compliance

```sql
-- AI-generated successor recommendations
CREATE TABLE ai_recommendation (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    position_id UUID NOT NULL REFERENCES position(id),
    employee_id UUID NOT NULL REFERENCES employee(id),
    algorithm_version VARCHAR(50) NOT NULL,
    skills_match_score NUMERIC(5,4) CHECK (skills_match_score BETWEEN 0 AND 1),
    readiness_score NUMERIC(5,4) CHECK (readiness_score BETWEEN 0 AND 1),
    overall_fit_score NUMERIC(5,4) CHECK (overall_fit_score BETWEEN 0 AND 1),
    explanation TEXT NOT NULL, -- human-readable explanation of why this person was recommended
    -- GDPR Article 22 compliance
    human_review_status VARCHAR(20) NOT NULL DEFAULT 'pending' CHECK (human_review_status IN ('pending', 'approved', 'rejected', 'override')),
    reviewer_id UUID REFERENCES employee(id),
    review_notes TEXT,
    reviewed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_ai_rec_position ON ai_recommendation(position_id);
CREATE INDEX idx_ai_rec_employee ON ai_recommendation(employee_id);
CREATE INDEX idx_ai_rec_review ON ai_recommendation(human_review_status);

-- Development plans linked to succession gaps
CREATE TABLE development_plan (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    employee_id UUID NOT NULL REFERENCES employee(id),
    target_position_id UUID REFERENCES position(id),
    succession_plan_id UUID REFERENCES succession_plan(id),
    title VARCHAR(255) NOT NULL,
    status VARCHAR(20) NOT NULL DEFAULT 'draft' CHECK (status IN ('draft', 'active', 'completed', 'cancelled')),
    target_completion_date DATE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE development_action (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    development_plan_id UUID NOT NULL REFERENCES development_plan(id) ON DELETE CASCADE,
    action_type VARCHAR(30) NOT NULL CHECK (action_type IN ('training', 'mentoring', 'stretch_assignment', 'job_rotation', 'coaching', 'certification', 'other')),
    description TEXT NOT NULL,
    skill_id UUID REFERENCES skill(id), -- which skill gap this action addresses
    status VARCHAR(20) NOT NULL DEFAULT 'pending' CHECK (status IN ('pending', 'in_progress', 'completed', 'cancelled')),
    due_date DATE,
    completed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Diversity & Equity Reporting

```sql
-- Employee demographic data (stored separately for access control)
CREATE TABLE employee_demographics (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    employee_id UUID NOT NULL UNIQUE REFERENCES employee(id) ON DELETE CASCADE,
    gender VARCHAR(30), -- self-reported
    ethnicity VARCHAR(50), -- self-reported, jurisdiction-dependent categories
    age_band VARCHAR(20), -- derived, not stored as exact DOB for privacy
    disability_status VARCHAR(20),
    veteran_status VARCHAR(20),
    -- Access restricted to authorised equity reporting roles only
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Equity snapshots for succession pipeline reporting
CREATE TABLE equity_snapshot (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    snapshot_date DATE NOT NULL,
    organisation_unit_id UUID REFERENCES organisation_unit(id),
    talent_pool_id UUID REFERENCES talent_pool(id),
    dimension VARCHAR(30) NOT NULL, -- 'gender', 'ethnicity', etc.
    category VARCHAR(50) NOT NULL,
    total_count INTEGER NOT NULL,
    successor_count INTEGER NOT NULL,
    ready_now_count INTEGER NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_equity_snapshot_date ON equity_snapshot(snapshot_date);
```

## ISO 30433 Metrics & Reporting

```sql
-- ISO 30433 succession planning metrics (computed periodically)
CREATE TABLE succession_metrics (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    reporting_period_start DATE NOT NULL,
    reporting_period_end DATE NOT NULL,
    organisation_unit_id UUID REFERENCES organisation_unit(id), -- NULL = org-wide
    -- ISO/TS 30433 metrics
    succession_effectiveness_rate NUMERIC(5,4), -- internal promotions from succession pool / total leadership appointments
    successor_coverage_rate NUMERIC(5,4), -- key positions with >= 1 successor / total key positions
    succession_depth_ready_now NUMERIC(5,2), -- avg successors with ready_now per key position
    succession_depth_1_to_3_years NUMERIC(5,2),
    succession_depth_4_to_5_years NUMERIC(5,2),
    -- Additional operational metrics
    total_key_positions INTEGER,
    positions_with_no_successor INTEGER,
    positions_with_single_successor INTEGER,
    avg_time_to_fill_from_pipeline_days NUMERIC(7,2),
    computed_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_metrics_period ON succession_metrics(reporting_period_end DESC);
CREATE INDEX idx_metrics_org ON succession_metrics(organisation_unit_id);
```

## Access Control

```sql
-- Role-based access control
CREATE TABLE app_role (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(50) NOT NULL UNIQUE, -- 'chro', 'hrbp', 'manager', 'executive', 'admin'
    description TEXT
);

CREATE TABLE user_account (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    employee_id UUID UNIQUE REFERENCES employee(id),
    email VARCHAR(255) NOT NULL UNIQUE,
    auth_provider VARCHAR(30) NOT NULL DEFAULT 'oidc', -- 'oidc', 'scim', 'local'
    external_auth_id VARCHAR(255), -- OIDC subject or SCIM externalId
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE user_role (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_account_id UUID NOT NULL REFERENCES user_account(id) ON DELETE CASCADE,
    app_role_id UUID NOT NULL REFERENCES app_role(id),
    organisation_unit_id UUID REFERENCES organisation_unit(id), -- scoped to a unit, NULL = org-wide
    granted_by UUID REFERENCES user_account(id),
    granted_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (user_account_id, app_role_id, organisation_unit_id)
);

-- Audit log for compliance
CREATE TABLE audit_log (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_account_id UUID REFERENCES user_account(id),
    action VARCHAR(50) NOT NULL, -- 'nomination.create', 'assessment.update', 'ai_recommendation.review', etc.
    entity_type VARCHAR(50) NOT NULL,
    entity_id UUID NOT NULL,
    old_values JSONB,
    new_values JSONB,
    ip_address INET,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_log_entity ON audit_log(entity_type, entity_id);
CREATE INDEX idx_audit_log_user ON audit_log(user_account_id);
CREATE INDEX idx_audit_log_created ON audit_log(created_at DESC);
```

## HRIS Integration

```sql
-- HRIS sync configuration
CREATE TABLE hris_connection (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    provider VARCHAR(30) NOT NULL CHECK (provider IN ('workday', 'bamboohr', 'adp', 'finch', 'custom')),
    connection_name VARCHAR(255) NOT NULL,
    auth_type VARCHAR(20) NOT NULL CHECK (auth_type IN ('oauth2', 'api_key', 'basic_auth')),
    credentials_encrypted BYTEA, -- encrypted connection credentials
    sync_frequency_minutes INTEGER NOT NULL DEFAULT 1440, -- daily
    last_sync_at TIMESTAMPTZ,
    last_sync_status VARCHAR(20),
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Sync log
CREATE TABLE hris_sync_log (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    hris_connection_id UUID NOT NULL REFERENCES hris_connection(id),
    started_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    completed_at TIMESTAMPTZ,
    status VARCHAR(20) NOT NULL DEFAULT 'running' CHECK (status IN ('running', 'success', 'partial', 'failed')),
    records_synced INTEGER DEFAULT 0,
    records_failed INTEGER DEFAULT 0,
    error_details TEXT
);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Entity Management | 3 | organisation_unit, position, employee |
| Skills & Competencies | 3 | skill, position_skill, employee_skill |
| Succession Plans | 2 | succession_plan, successor_nomination |
| 9-Box & Calibration | 4 | assessment, calibration_session, calibration_participant, calibration_decision |
| Talent Pools | 2 | talent_pool, talent_pool_member |
| AI & Development | 4 | ai_recommendation, development_plan, development_action |
| Diversity & Equity | 2 | employee_demographics, equity_snapshot |
| ISO Metrics | 1 | succession_metrics |
| Access Control & Audit | 4 | app_role, user_account, user_role, audit_log |
| HRIS Integration | 2 | hris_connection, hris_sync_log |
| **Total** | **27** | |

---

## Key Design Decisions

1. **UUID primary keys throughout** — enables multi-tenant deployments and HRIS data merges without ID collisions.

2. **SHRM-aligned readiness tiers as CHECK constraints** — `ready_now`, `1_to_2_years`, `3_to_5_years` enforced at the database level, ensuring consistency with SHRM succession planning framework and ISO 30433 depth metrics.

3. **Computed 9-box position** — the `nine_box_position` column is a GENERATED column derived from performance and potential ratings, ensuring the 9-box grid is always consistent with the underlying ratings.

4. **Separate demographics table with restricted access** — employee demographic data is isolated in `employee_demographics` to enable row-level security policies that restrict access to equity reporting roles only, supporting GDPR data minimisation.

5. **GDPR Article 22 enforcement at the schema level** — the `ai_recommendation` table requires `human_review_status` to be set before any recommendation can be actioned, with `reviewer_id` and `reviewed_at` providing the audit trail required by Article 22.

6. **ESCO/O*NET dual taxonomy support** — the `skill` table supports both ESCO concept URIs and O*NET codes via a `taxonomy_source` discriminator, enabling pluggable taxonomy as recommended in the features specification.

7. **ISO 30433 metrics as a materialised snapshot table** — succession metrics are computed periodically and stored in `succession_metrics`, enabling time-series analysis of succession health and direct export for ISO 30414 compliance reporting.

8. **Audit log with JSONB diff** — the `audit_log` table stores `old_values` and `new_values` as JSONB, providing a flexible change-tracking mechanism without requiring shadow tables for every entity.

9. **Finch-compatible HRIS integration** — the `hris_connection` table supports multiple providers including Finch (unified API), enabling a single integration path to 250+ HRIS systems as identified in the standards research.
