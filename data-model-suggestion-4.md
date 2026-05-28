# Data Model Suggestion 4: Graph-Relational Hybrid

> Project: Succession Planning Tool · Created: 2026-05-12

## Philosophy

This model combines a conventional relational layer for operational CRUD (positions, employees, succession plans, RBAC) with a property graph layer for relationship-heavy queries: skills adjacency, successor path finding, organisational influence networks, and bench strength analysis across complex hierarchies. The graph layer is implemented as a lightweight property graph within PostgreSQL using `graph_node` and `graph_edge` tables with JSONB properties, avoiding the operational complexity of a separate graph database while enabling graph traversal via recursive CTEs.

Succession planning is fundamentally a graph problem. The core questions — "Who could succeed the CFO?", "What is the shortest development path from Employee A to Position B?", "Which positions have cascading single-point-of-failure risk?" — are graph traversal queries. Skills adjacency (matching employees to positions based on ESCO/O*NET skill graphs), career path mapping, and conflict-of-interest analysis (an employee cannot succeed their spouse's role) all benefit from graph-native query patterns.

This approach is inspired by LinkedIn's Skills Graph (which powers job recommendations via skills adjacency) and by how talent intelligence platforms like Eightfold AI model workforce capabilities as a graph of skills, experiences, and potential transitions. The relational layer handles structured operations (HRIS sync, RBAC, calibration sessions, ISO 30433 metrics), while the graph layer handles the exploratory, relationship-dense queries that distinguish an AI-native succession tool from a traditional 9-box grid.

**Best for:** Organisations that prioritise AI-driven successor identification, skills-based career pathing, and the ability to model complex workforce relationships (cross-functional exposure, mentoring networks, reporting chains) as first-class queryable structures.

**Trade-offs:**
- Pro: Skills adjacency queries ("employees within 2 skill hops of this position") are natural and fast
- Pro: Cascading bench strength risk analysis ("if these 3 people leave, what positions have no coverage?") is a graph traversal, not a complex multi-JOIN
- Pro: Career path modelling ("all paths from Software Engineer to VP Engineering") is a graph shortest-path query
- Pro: Conflict-of-interest and bias detection (e.g., "successor was nominated by their manager who is also the incumbent") are edge relationship queries
- Pro: Graph stays in PostgreSQL — no separate graph database to operate
- Con: Recursive CTEs for deep graph traversals can be slow without careful depth limits and indexing
- Con: Dual model (relational + graph) increases conceptual complexity for developers
- Con: Graph data must be kept in sync with relational data (denormalization maintenance)
- Con: Lack of native graph query language (no Cypher/GQL) — queries use SQL recursive CTEs which are verbose
- Con: Not a full-featured graph database — some advanced graph algorithms (PageRank, community detection) require application-level implementation

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ESCO Classification | ESCO skills taxonomy is loaded as graph nodes with `broaderSkill`/`narrowerSkill` edges, enabling skills adjacency traversal |
| O*NET | O*NET occupations and skills are alternate graph nodes connected via `requiresSkill` edges |
| ISO/TS 30433:2021 | Succession metrics computed from relational tables; graph layer used to identify cascading risk not captured by simple counts |
| ISO 30414:2025 | Standard metrics from relational layer; graph-derived insights (network centrality, skills coverage breadth) supplement disclosures |
| SHRM Framework | Readiness tiers stored relationally; graph models the development pathway from current state to readiness |
| GDPR Article 22 | AI recommendations that traverse the graph include explainability via the path taken (which skills matched, which adjacencies were traversed) |
| HR Open Standards | Relational entities export directly to HR-JSON; graph relationships export as linked resource references |

---

## Graph Layer

```sql
-- Property graph implemented in PostgreSQL
-- Inspired by Apache AGE and adjacency-list graph patterns

-- Graph nodes: represent entities that participate in relationships
CREATE TABLE graph_node (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    node_type VARCHAR(50) NOT NULL,
    -- node_type values:
    -- 'employee', 'position', 'skill', 'organisation_unit', 'talent_pool',
    -- 'occupation' (ESCO/O*NET), 'competency_area'
    entity_id UUID NOT NULL, -- FK to the corresponding relational table
    label VARCHAR(255) NOT NULL, -- human-readable label for display
    properties JSONB NOT NULL DEFAULT '{}',
    -- properties examples:
    -- For employee: { "tenure_years": 3.5, "nine_box": 9, "flight_risk": "low" }
    -- For skill: { "esco_uri": "http://...", "taxonomy": "esco", "reusability": "transversal" }
    -- For position: { "level": "VP", "is_key": true, "criticality": "mission_critical" }
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (node_type, entity_id)
);

CREATE INDEX idx_gnode_type ON graph_node(node_type);
CREATE INDEX idx_gnode_entity ON graph_node(entity_id);
CREATE INDEX idx_gnode_props ON graph_node USING GIN (properties);

-- Graph edges: relationships between nodes
CREATE TABLE graph_edge (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    source_node_id UUID NOT NULL REFERENCES graph_node(id) ON DELETE CASCADE,
    target_node_id UUID NOT NULL REFERENCES graph_node(id) ON DELETE CASCADE,
    edge_type VARCHAR(50) NOT NULL,
    -- edge_type values:
    -- 'has_skill' (employee -> skill, weight = proficiency)
    -- 'requires_skill' (position -> skill, weight = importance)
    -- 'reports_to' (employee -> employee)
    -- 'nominated_for' (employee -> position, properties include readiness_tier)
    -- 'broader_skill' (skill -> skill, ESCO hierarchy)
    -- 'related_skill' (skill -> skill, ESCO/O*NET adjacency)
    -- 'occupation_skill' (occupation -> skill)
    -- 'member_of' (employee -> talent_pool)
    -- 'belongs_to' (position -> organisation_unit)
    -- 'mentored_by' (employee -> employee)
    -- 'career_transition' (position -> position, historical transition data)
    weight NUMERIC(5,2) DEFAULT 1.0, -- relationship strength / proficiency / importance
    properties JSONB NOT NULL DEFAULT '{}',
    -- properties examples:
    -- For has_skill: { "proficiency": 4, "source": "manager_assessed", "assessed_at": "2026-03-01" }
    -- For nominated_for: { "readiness_tier": "ready_now", "rank": 1, "nominated_by": "uuid" }
    -- For career_transition: { "transition_count": 15, "avg_months": 24, "success_rate": 0.82 }
    -- For related_skill: { "adjacency_score": 0.75, "source": "esco" }
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_gedge_source ON graph_edge(source_node_id);
CREATE INDEX idx_gedge_target ON graph_edge(target_node_id);
CREATE INDEX idx_gedge_type ON graph_edge(edge_type);
CREATE INDEX idx_gedge_source_type ON graph_edge(source_node_id, edge_type);
CREATE INDEX idx_gedge_target_type ON graph_edge(target_node_id, edge_type);
CREATE INDEX idx_gedge_props ON graph_edge USING GIN (properties);
```

## Relational Layer (Operational CRUD)

```sql
CREATE TABLE tenant (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    slug VARCHAR(100) NOT NULL UNIQUE,
    config JSONB NOT NULL DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE organisation_unit (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenant(id),
    parent_id UUID REFERENCES organisation_unit(id),
    name VARCHAR(255) NOT NULL,
    unit_type VARCHAR(50) NOT NULL,
    country_code CHAR(2),
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
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_position_tenant ON position(tenant_id);
CREATE INDEX idx_position_key ON position(is_key_position) WHERE is_key_position = true;

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
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_employee_tenant ON employee(tenant_id);
CREATE INDEX idx_employee_position ON employee(current_position_id);

CREATE TABLE skill (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    taxonomy_source VARCHAR(20) NOT NULL,
    external_uri VARCHAR(500),
    skill_type VARCHAR(30) NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_skill_taxonomy ON skill(taxonomy_source, external_uri);

-- Succession plans and nominations (relational for CRUD operations)
CREATE TABLE succession_plan (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenant(id),
    position_id UUID NOT NULL REFERENCES position(id),
    owner_id UUID NOT NULL REFERENCES employee(id),
    status VARCHAR(20) NOT NULL DEFAULT 'draft',
    last_reviewed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE successor_nomination (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    succession_plan_id UUID NOT NULL REFERENCES succession_plan(id) ON DELETE CASCADE,
    employee_id UUID NOT NULL REFERENCES employee(id),
    nominated_by UUID NOT NULL REFERENCES employee(id),
    readiness_tier VARCHAR(30) NOT NULL,
    rank_order INTEGER,
    nomination_status VARCHAR(20) NOT NULL DEFAULT 'nominated',
    risk_of_loss VARCHAR(20),
    impact_of_loss VARCHAR(20),
    nominated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (succession_plan_id, employee_id)
);

-- Assessments
CREATE TABLE assessment (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenant(id),
    employee_id UUID NOT NULL REFERENCES employee(id),
    assessment_period_end DATE NOT NULL,
    performance_rating INTEGER NOT NULL CHECK (performance_rating BETWEEN 1 AND 3),
    potential_rating INTEGER NOT NULL CHECK (potential_rating BETWEEN 1 AND 3),
    source VARCHAR(30) NOT NULL,
    assessed_by UUID REFERENCES employee(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_assessment_employee ON assessment(employee_id);

-- Talent pools
CREATE TABLE talent_pool (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenant(id),
    name VARCHAR(255) NOT NULL,
    pool_type VARCHAR(30) NOT NULL,
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
    readiness_tier VARCHAR(30),
    added_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (talent_pool_id, employee_id)
);

-- AI recommendations
CREATE TABLE ai_recommendation (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenant(id),
    position_id UUID NOT NULL REFERENCES position(id),
    employee_id UUID NOT NULL REFERENCES employee(id),
    algorithm_version VARCHAR(50) NOT NULL,
    overall_fit_score NUMERIC(5,4),
    skills_match_score NUMERIC(5,4),
    graph_path_score NUMERIC(5,4), -- score derived from graph traversal
    explanation TEXT NOT NULL,
    graph_path JSONB, -- the actual graph path that produced the recommendation
    -- graph_path example:
    -- {
    --   "path": ["employee-uuid", "skill-uuid-1", "skill-uuid-2", "position-uuid"],
    --   "edge_types": ["has_skill", "related_skill", "requires_skill"],
    --   "total_hops": 3,
    --   "adjacency_scores": [0.85, 0.72, 0.91]
    -- }
    human_review_status VARCHAR(20) NOT NULL DEFAULT 'pending',
    reviewer_id UUID REFERENCES employee(id),
    reviewed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_ai_rec_position ON ai_recommendation(position_id);
CREATE INDEX idx_ai_rec_review ON ai_recommendation(human_review_status);

-- ISO 30433 metrics
CREATE TABLE succession_metrics (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenant(id),
    reporting_period_end DATE NOT NULL,
    organisation_unit_id UUID REFERENCES organisation_unit(id),
    succession_effectiveness_rate NUMERIC(5,4),
    successor_coverage_rate NUMERIC(5,4),
    succession_depth_ready_now NUMERIC(5,2),
    succession_depth_1_to_3_years NUMERIC(5,2),
    succession_depth_4_to_5_years NUMERIC(5,2),
    -- Graph-derived supplementary metrics
    avg_skills_distance NUMERIC(5,2), -- average graph hops between successors and target positions
    cascading_risk_positions INTEGER, -- positions where losing 1 successor affects other positions' bench
    computed_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Audit log
CREATE TABLE audit_log (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenant(id),
    user_id UUID,
    action VARCHAR(50) NOT NULL,
    entity_type VARCHAR(50) NOT NULL,
    entity_id UUID NOT NULL,
    changes JSONB,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
CREATE INDEX idx_audit_created ON audit_log(created_at DESC);

-- Access control
CREATE TABLE user_account (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenant(id),
    employee_id UUID REFERENCES employee(id),
    email VARCHAR(255) NOT NULL,
    auth_provider VARCHAR(30) NOT NULL DEFAULT 'oidc',
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, email)
);

CREATE TABLE app_role (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(50) NOT NULL UNIQUE,
    description TEXT
);

CREATE TABLE user_role (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_account_id UUID NOT NULL REFERENCES user_account(id) ON DELETE CASCADE,
    app_role_id UUID NOT NULL REFERENCES app_role(id),
    organisation_unit_id UUID REFERENCES organisation_unit(id),
    UNIQUE (user_account_id, app_role_id, organisation_unit_id)
);

-- HRIS connection
CREATE TABLE hris_connection (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenant(id),
    provider VARCHAR(30) NOT NULL,
    connection_name VARCHAR(255) NOT NULL,
    auth_type VARCHAR(20) NOT NULL,
    credentials_encrypted BYTEA,
    last_sync_at TIMESTAMPTZ,
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Example Graph Queries

### Find successor candidates via skills adjacency (within 2 hops)

```sql
-- Find employees whose skills are within 2 adjacency hops of the skills required for VP Finance
WITH target_skills AS (
    -- Skills required by the target position
    SELECT ge.target_node_id AS skill_node_id
    FROM graph_node gn
    JOIN graph_edge ge ON ge.source_node_id = gn.id AND ge.edge_type = 'requires_skill'
    WHERE gn.node_type = 'position' AND gn.entity_id = '<vp-finance-uuid>'
),
adjacent_skills AS (
    -- Skills within 1 hop of the target skills (via related_skill edges)
    SELECT DISTINCT ge.target_node_id AS skill_node_id, 1 AS hops
    FROM target_skills ts
    JOIN graph_edge ge ON ge.source_node_id = ts.skill_node_id AND ge.edge_type = 'related_skill'
    UNION ALL
    -- Skills within 2 hops
    SELECT DISTINCT ge2.target_node_id AS skill_node_id, 2 AS hops
    FROM target_skills ts
    JOIN graph_edge ge1 ON ge1.source_node_id = ts.skill_node_id AND ge1.edge_type = 'related_skill'
    JOIN graph_edge ge2 ON ge2.source_node_id = ge1.target_node_id AND ge2.edge_type = 'related_skill'
),
all_relevant_skills AS (
    SELECT skill_node_id, 0 AS hops FROM target_skills
    UNION ALL
    SELECT skill_node_id, hops FROM adjacent_skills
),
candidate_scores AS (
    SELECT
        emp_node.entity_id AS employee_id,
        emp_node.label AS employee_name,
        COUNT(DISTINCT ars.skill_node_id) AS matching_skills,
        (SELECT COUNT(*) FROM target_skills) AS required_skills,
        AVG(CASE WHEN ars.hops = 0 THEN 1.0 WHEN ars.hops = 1 THEN 0.7 ELSE 0.4 END) AS adjacency_score
    FROM graph_node emp_node
    JOIN graph_edge has_skill ON has_skill.source_node_id = emp_node.id AND has_skill.edge_type = 'has_skill'
    JOIN all_relevant_skills ars ON ars.skill_node_id = has_skill.target_node_id
    WHERE emp_node.node_type = 'employee'
    GROUP BY emp_node.entity_id, emp_node.label
)
SELECT
    employee_id,
    employee_name,
    matching_skills,
    required_skills,
    ROUND(adjacency_score, 2) AS adjacency_score,
    ROUND(matching_skills::NUMERIC / NULLIF(required_skills, 0), 2) AS coverage_ratio
FROM candidate_scores
WHERE matching_skills >= required_skills * 0.5 -- at least 50% coverage
ORDER BY adjacency_score DESC, matching_skills DESC
LIMIT 20;
```

### Cascading bench strength risk analysis

```sql
-- If employee X leaves, which other succession plans lose a successor?
-- (Identifies cascading single-point-of-failure risk)
WITH departing_employee AS (
    SELECT id FROM graph_node WHERE node_type = 'employee' AND entity_id = '<employee-uuid>'
),
affected_positions AS (
    -- Positions where this employee is nominated
    SELECT
        ge.target_node_id AS position_node_id,
        gn.entity_id AS position_id,
        gn.label AS position_title,
        ge.properties->>'readiness_tier' AS readiness_tier
    FROM departing_employee de
    JOIN graph_edge ge ON ge.source_node_id = de.id AND ge.edge_type = 'nominated_for'
    JOIN graph_node gn ON gn.id = ge.target_node_id
),
bench_after_departure AS (
    -- For each affected position, count remaining successors
    SELECT
        ap.position_id,
        ap.position_title,
        ap.readiness_tier AS departing_readiness,
        COUNT(ge2.id) - 1 AS remaining_successors -- subtract the departing employee
    FROM affected_positions ap
    JOIN graph_edge ge2 ON ge2.target_node_id = ap.position_node_id AND ge2.edge_type = 'nominated_for'
    GROUP BY ap.position_id, ap.position_title, ap.readiness_tier
)
SELECT
    position_id,
    position_title,
    departing_readiness,
    remaining_successors,
    CASE
        WHEN remaining_successors = 0 THEN 'CRITICAL: No successors remain'
        WHEN remaining_successors = 1 THEN 'WARNING: Single successor remaining'
        ELSE 'OK'
    END AS risk_level
FROM bench_after_departure
ORDER BY remaining_successors ASC;
```

### Career path discovery (shortest development path)

```sql
-- Find the shortest skills development path from an employee's current position to a target position
WITH RECURSIVE career_path AS (
    -- Start from the employee's current position
    SELECT
        gn.id AS current_node,
        gn.entity_id AS position_id,
        gn.label AS position_title,
        ARRAY[gn.id] AS path,
        ARRAY[gn.label] AS path_labels,
        0 AS depth,
        0.0::NUMERIC AS total_transition_weight
    FROM graph_node gn
    WHERE gn.node_type = 'position' AND gn.entity_id = '<current-position-uuid>'

    UNION ALL

    -- Traverse career_transition edges
    SELECT
        ge.target_node_id AS current_node,
        gn2.entity_id AS position_id,
        gn2.label AS position_title,
        cp.path || ge.target_node_id,
        cp.path_labels || gn2.label,
        cp.depth + 1,
        cp.total_transition_weight + COALESCE(ge.weight, 1.0)
    FROM career_path cp
    JOIN graph_edge ge ON ge.source_node_id = cp.current_node AND ge.edge_type = 'career_transition'
    JOIN graph_node gn2 ON gn2.id = ge.target_node_id
    WHERE cp.depth < 5 -- limit path length
      AND ge.target_node_id != ALL(cp.path) -- prevent cycles
)
SELECT
    path_labels AS career_path,
    depth AS transitions,
    total_transition_weight
FROM career_path
WHERE position_id = '<target-position-uuid>'
ORDER BY total_transition_weight ASC, depth ASC
LIMIT 5;
```

### Skills gap analysis via graph

```sql
-- For a specific employee and target position, find the skills gap
-- (skills the position requires that the employee doesn't have or has at insufficient proficiency)
SELECT
    s_node.label AS skill_name,
    s_node.properties->>'esco_uri' AS esco_uri,
    req_edge.weight AS required_proficiency,
    COALESCE(has_edge.weight, 0) AS current_proficiency,
    req_edge.weight - COALESCE(has_edge.weight, 0) AS gap
FROM graph_node pos_node
JOIN graph_edge req_edge ON req_edge.source_node_id = pos_node.id AND req_edge.edge_type = 'requires_skill'
JOIN graph_node s_node ON s_node.id = req_edge.target_node_id
LEFT JOIN graph_edge has_edge ON has_edge.target_node_id = s_node.id
    AND has_edge.edge_type = 'has_skill'
    AND has_edge.source_node_id = (
        SELECT id FROM graph_node WHERE node_type = 'employee' AND entity_id = '<employee-uuid>'
    )
WHERE pos_node.node_type = 'position' AND pos_node.entity_id = '<target-position-uuid>'
  AND (has_edge.weight IS NULL OR has_edge.weight < req_edge.weight)
ORDER BY gap DESC;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Graph Layer | 2 | graph_node, graph_edge (flexible property graph) |
| Core Entities | 5 | tenant, organisation_unit, position, employee, skill |
| Succession | 2 | succession_plan, successor_nomination |
| Assessment | 1 | assessment |
| Talent Pools | 2 | talent_pool, talent_pool_member |
| AI | 1 | ai_recommendation (with graph_path JSONB) |
| Metrics & Audit | 2 | succession_metrics, audit_log |
| Access Control | 3 | user_account, app_role, user_role |
| Integration | 1 | hris_connection |
| **Total** | **19** | 2 graph tables + 17 relational tables |

---

## Key Design Decisions

1. **Property graph in PostgreSQL, not a separate graph database** — using `graph_node` and `graph_edge` tables avoids the operational burden of running Neo4j alongside PostgreSQL. For the query patterns succession planning requires (skills adjacency within 2-3 hops, career path finding within 5 transitions, bench strength cascading analysis), PostgreSQL recursive CTEs perform adequately for workforces up to ~50,000 employees.

2. **Graph complements relational, not replaces** — CRUD operations (create succession plan, add nomination, record assessment) use the relational tables. The graph layer is a derived, queryable view of relationships that powers AI recommendations, skills matching, and scenario analysis. If the graph layer is unavailable, the tool still functions for basic succession planning.

3. **ESCO/O*NET loaded as graph nodes with adjacency edges** — the ESCO skills taxonomy (13,485 skills with `broaderSkill`/`narrowerSkill`/`relatedSkill` relationships) is a natural graph structure. Loading it as nodes and edges enables skills adjacency queries that would be impossible with flat relational tables.

4. **Career transition edges built from historical data** — `career_transition` edges between position nodes capture how often employees have historically moved between roles, the average time for the transition, and the success rate. This data powers AI-driven career path recommendations.

5. **Graph path stored in AI recommendations** — the `graph_path` JSONB column on `ai_recommendation` stores the actual traversal path the algorithm took, providing GDPR Article 22 explainability: "This employee was recommended because they have Skill A (proficiency 4), which is adjacent to Skill B (adjacency score 0.75), which Position X requires."

6. **Weight-based edge scoring** — the `weight` column on `graph_edge` serves different purposes per edge type: proficiency level for `has_skill`, importance for `requires_skill`, adjacency score for `related_skill`, and transition frequency for `career_transition`. This enables weighted graph traversal for more nuanced matching.

7. **Graph-derived metrics supplement ISO 30433** — beyond standard ISO metrics (computed from relational data), the graph layer contributes `avg_skills_distance` (how far successors are from target position requirements in the skills graph) and `cascading_risk_positions` (positions where a single departure affects multiple succession plans), stored in `succession_metrics`.

8. **Dual sync: HRIS to relational, relational to graph** — HRIS data flows into relational tables first, then a graph sync process creates/updates corresponding graph nodes and edges. This two-phase approach keeps the HRIS integration simple while ensuring the graph stays current.

9. **Depth-limited recursive CTEs** — all graph traversal queries include a `depth < N` guard to prevent runaway queries in dense skill graphs. For succession planning, 3-5 hops is sufficient for skills adjacency and career path discovery.
