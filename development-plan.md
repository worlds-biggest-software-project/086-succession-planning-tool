# Succession Planning Tool — Phased Development Plan

> Project: 086-succession-planning-tool
> Generated: 2026-05-25
> Status: Research-informed development blueprint

---

## Table of Contents

1. [Technology Decisions](#technology-decisions)
2. [Project Structure](#project-structure)
3. [Phase Dependency Graph](#phase-dependency-graph)
4. [Phase 1: Foundation & Core Data Model](#phase-1-foundation--core-data-model)
5. [Phase 2: Authentication, RBAC & Audit](#phase-2-authentication-rbac--audit)
6. [Phase 3: Succession Plans, Nominations & 9-Box Grid](#phase-3-succession-plans-nominations--9-box-grid)
7. [Phase 4: Talent Pools & Calibration Sessions](#phase-4-talent-pools--calibration-sessions)
8. [Phase 5: HRIS Integration Layer](#phase-5-hris-integration-layer)
9. [Phase 6: Bench Strength Dashboard & Risk Reporting](#phase-6-bench-strength-dashboard--risk-reporting)
10. [Phase 7: Skills Taxonomy & Graph Layer](#phase-7-skills-taxonomy--graph-layer)
11. [Phase 8: AI Successor Identification](#phase-8-ai-successor-identification)
12. [Phase 9: Equity Lens & Diversity Reporting](#phase-9-equity-lens--diversity-reporting)
13. [Phase 10: Board-Ready Reports & ISO 30433 Metrics](#phase-10-board-ready-reports--iso-30433-metrics)
14. [Phase 11: Scenario Modeling & NL Queries](#phase-11-scenario-modeling--nl-queries)
15. [Phase 12: Self-Hosted Deployment & SOC 2 Readiness](#phase-12-self-hosted-deployment--soc-2-readiness)
16. [Definition of Done (Global)](#definition-of-done-global)

---

## Technology Decisions

### Database: PostgreSQL 16+

**Rationale:** All four data model suggestions are PostgreSQL-native. The project requires JSONB (for tenant-configurable fields), GIN indexes (for skills search), recursive CTEs (for graph traversal and org hierarchy queries), and generated columns (for computed 9-box positions). PostgreSQL is the only open-source RDBMS that provides all four natively. It also supports row-level security for demographic data isolation and temporal table patterns needed for historical bench strength analysis.

**Data model choice: Hybrid Relational + JSONB (Suggestion 3) as the primary model, with the Graph Layer (Suggestion 4) added in Phase 7.**

Rationale for this combination:
- Suggestion 1 (fully normalized) has too many tables (~27) for an MVP targeting a small team, and schema changes require migrations for every tenant customization.
- Suggestion 2 (event-sourced CQRS) adds implementation complexity that is not justified until the user base demands full temporal auditability; it can be adopted later by adding an event store alongside the relational model.
- Suggestion 3 (hybrid JSONB) gives 17 tables, multi-tenant support via tenant config, and absorbs jurisdiction-specific variation (diversity categories, readiness tier labels, custom fields) without migrations. This is the right balance for an MVP.
- Suggestion 4 (graph layer) provides the skills adjacency and career path queries that power AI successor identification. The `graph_node`/`graph_edge` tables are added in Phase 7 as a derived layer over the relational model — the two coexist naturally.

### Backend: Node.js 22 LTS + TypeScript 5.5 + Fastify

**Rationale:** The mid-market target segment (200-2,000 employees) requires a tech stack that is accessible to a broad contributor base. TypeScript with Fastify offers strong type safety, OpenAPI schema generation from route definitions (via `@fastify/swagger`), and first-class JSON Schema validation — critical for validating tenant-configurable JSONB payloads. Fastify's plugin architecture maps cleanly to the modular domain (succession, calibration, talent pools, AI, HRIS integration). Node.js also has mature SDKs for BambooHR, ADP, and Finch.

### ORM / Query Builder: Drizzle ORM

**Rationale:** Drizzle provides type-safe SQL with explicit control over queries (no hidden N+1 problems), first-class PostgreSQL JSONB operator support, and migration generation from TypeScript schema definitions. Unlike Prisma, Drizzle does not abstract away SQL — important for the graph traversal queries and recursive CTEs in Phases 7-8.

### Frontend: Next.js 15 (App Router) + React 19 + Tailwind CSS + shadcn/ui

**Rationale:** Next.js App Router provides server components (for data-heavy dashboards like bench strength heat maps), server actions (for succession plan mutations), and API routes (for HRIS webhook receivers). shadcn/ui is unstyled and composable — essential for the 9-box grid, org chart, and calibration session interfaces that require custom interaction patterns. Tailwind CSS enables rapid iteration on the dense data-display UIs this domain requires.

### AI / ML: Python microservice (FastAPI + scikit-learn + sentence-transformers)

**Rationale:** AI successor identification (Phase 8) requires skills embedding generation, cosine similarity computation, and readiness scoring from heterogeneous signals. Python's ML ecosystem is unmatched for these tasks. Packaging the AI logic as a separate FastAPI microservice keeps the Node.js monolith simple and allows independent scaling and model versioning. Communication between the Node.js backend and Python AI service uses a REST API with OpenAPI contract.

### Skills Taxonomy: ESCO API + O*NET Web Services (pluggable)

**Rationale:** Research identifies ESCO (13,939 skills, 28 languages, free EU open data licence) and O*NET (900+ occupations, free for OSS) as the two credible open taxonomies. Eightfold AI holds patents on proprietary skills inference — using open taxonomies avoids IP risk. The architecture provides a pluggable taxonomy adapter so deployments choose ESCO, O*NET, or both.

### Authentication: OpenID Connect (OIDC) via NextAuth.js v5

**Rationale:** Enterprise buyers require SSO. OIDC is supported by all major identity providers (Okta, Azure AD, Google Workspace, Keycloak for self-hosted). NextAuth.js v5 provides adapter-based OIDC integration with session management and CSRF protection.

### Containerisation: Docker + Docker Compose (dev), Helm chart (production)

**Rationale:** Self-hosted deployment is a stated requirement for data-sovereign organisations. Docker Compose enables single-command local development and testing. A Helm chart enables Kubernetes deployment for production cloud and on-premises environments.

### CI/CD: GitHub Actions

**Rationale:** Open-source project hosted on GitHub; Actions provides free CI for public repositories with matrix testing (Node 22, PostgreSQL 16), container build/push, and deployment automation.

---

## Project Structure

```
succession-planning-tool/
├── .github/
│   └── workflows/
│       ├── ci.yml                    # Lint, type-check, unit tests, integration tests
│       ├── e2e.yml                   # Playwright E2E tests
│       └── release.yml               # Docker build, Helm package, GitHub release
├── apps/
│   ├── web/                          # Next.js 15 frontend
│   │   ├── src/
│   │   │   ├── app/                  # App Router pages and layouts
│   │   │   │   ├── (auth)/           # Login, SSO callback
│   │   │   │   ├── (dashboard)/      # Main application shell
│   │   │   │   │   ├── succession/   # Succession plan CRUD, org chart
│   │   │   │   │   ├── calibration/  # 9-box grid, calibration sessions
│   │   │   │   │   ├── talent-pools/ # Pool management
│   │   │   │   │   ├── bench/        # Bench strength dashboard
│   │   │   │   │   ├── reports/      # Board reports, ISO metrics
│   │   │   │   │   ├── equity/       # Diversity lens
│   │   │   │   │   ├── ai/           # AI recommendations, NL queries
│   │   │   │   │   └── settings/     # HRIS connections, tenant config
│   │   │   │   └── api/              # API routes (webhooks, HRIS callbacks)
│   │   │   ├── components/           # Shared UI components
│   │   │   │   ├── nine-box/         # 9-box grid widget
│   │   │   │   ├── org-chart/        # Succession org chart
│   │   │   │   ├── heat-map/         # Bench strength heat map
│   │   │   │   └── data-table/       # Configurable data tables
│   │   │   └── lib/                  # Client utilities, API client
│   │   ├── public/
│   │   ├── next.config.ts
│   │   ├── tailwind.config.ts
│   │   └── package.json
│   └── ai-service/                   # Python FastAPI AI microservice
│       ├── src/
│       │   ├── embeddings/           # Skills embedding generation
│       │   ├── matching/             # Successor matching algorithms
│       │   ├── readiness/            # Readiness scoring engine
│       │   ├── scenarios/            # Bench strength scenario modeling
│       │   ├── nlp/                  # Natural language query processing
│       │   └── api/                  # FastAPI routes
│       ├── tests/
│       ├── pyproject.toml
│       └── Dockerfile
├── packages/
│   ├── db/                           # Drizzle schema, migrations, seed
│   │   ├── src/
│   │   │   ├── schema/               # Drizzle table definitions
│   │   │   │   ├── tenant.ts
│   │   │   │   ├── organisation.ts
│   │   │   │   ├── employee.ts
│   │   │   │   ├── position.ts
│   │   │   │   ├── skill.ts
│   │   │   │   ├── succession.ts
│   │   │   │   ├── assessment.ts
│   │   │   │   ├── calibration.ts
│   │   │   │   ├── talent-pool.ts
│   │   │   │   ├── ai-recommendation.ts
│   │   │   │   ├── graph.ts          # graph_node, graph_edge (Phase 7)
│   │   │   │   ├── metrics.ts
│   │   │   │   ├── audit.ts
│   │   │   │   ├── auth.ts
│   │   │   │   └── hris.ts
│   │   │   ├── migrations/
│   │   │   └── seed/
│   │   │       ├── demo-tenant.ts
│   │   │       ├── esco-skills.ts    # ESCO taxonomy loader
│   │   │       └── onet-skills.ts    # O*NET taxonomy loader
│   │   └── package.json
│   ├── api/                          # Fastify API server
│   │   ├── src/
│   │   │   ├── routes/
│   │   │   │   ├── succession/
│   │   │   │   ├── calibration/
│   │   │   │   ├── talent-pools/
│   │   │   │   ├── assessments/
│   │   │   │   ├── employees/
│   │   │   │   ├── positions/
│   │   │   │   ├── ai/
│   │   │   │   ├── reports/
│   │   │   │   ├── hris/
│   │   │   │   └── admin/
│   │   │   ├── plugins/              # Fastify plugins (auth, rbac, audit)
│   │   │   ├── services/             # Business logic layer
│   │   │   └── lib/                  # Shared utilities
│   │   ├── tests/
│   │   └── package.json
│   └── shared/                       # Shared types, constants, validators
│       ├── src/
│       │   ├── types/
│       │   ├── validators/           # Zod schemas for JSONB validation
│       │   └── constants/
│       └── package.json
├── deploy/
│   ├── docker-compose.yml            # Local dev: PostgreSQL, API, web, AI service
│   ├── docker-compose.prod.yml
│   ├── helm/
│   │   └── succession-tool/          # Helm chart for Kubernetes
│   └── Dockerfile.web
│   └── Dockerfile.api
├── docs/
│   ├── openapi.yaml                  # Generated OpenAPI spec
│   └── architecture.md
├── e2e/                              # Playwright E2E tests
├── turbo.json                        # Turborepo config
├── package.json                      # Root workspace
└── README.md
```

**Monorepo rationale:** Turborepo manages the `apps/web`, `apps/ai-service`, `packages/db`, `packages/api`, and `packages/shared` workspaces. This keeps the database schema, API server, and frontend in lockstep while allowing the Python AI service to have its own build/test lifecycle.

---

## Phase Dependency Graph

```
Phase 1: Foundation & Core Data Model
  │
  ├──▸ Phase 2: Authentication, RBAC & Audit
  │      │
  │      ├──▸ Phase 3: Succession Plans, Nominations & 9-Box Grid
  │      │      │
  │      │      ├──▸ Phase 4: Talent Pools & Calibration Sessions
  │      │      │      │
  │      │      │      ├──▸ Phase 6: Bench Strength Dashboard & Risk Reporting
  │      │      │      │      │
  │      │      │      │      ├──▸ Phase 9: Equity Lens & Diversity Reporting
  │      │      │      │      │      │
  │      │      │      │      │      └──▸ Phase 10: Board-Ready Reports & ISO 30433
  │      │      │      │      │
  │      │      │      │      └──▸ Phase 11: Scenario Modeling & NL Queries ◂── Phase 8
  │      │      │      │
  │      │      │      └──▸ Phase 7: Skills Taxonomy & Graph Layer
  │      │      │             │
  │      │      │             └──▸ Phase 8: AI Successor Identification
  │      │      │
  │      │      └──▸ Phase 5: HRIS Integration Layer
  │      │
  │      └──▸ Phase 12: Self-Hosted Deployment & SOC 2 Readiness ◂── Phase 10
  │
  (All phases depend on Phase 1 implicitly)
```

**Critical path:** 1 → 2 → 3 → 4 → 7 → 8 → 11

**Parallel tracks after Phase 3:**
- Track A (Integration): Phase 5 (HRIS) can proceed in parallel with Phases 4 and 6
- Track B (AI): Phase 7 → 8 can proceed in parallel with Phase 6 → 9
- Track C (Compliance): Phase 12 can begin after Phase 10, but infrastructure work (Docker, Helm) can start alongside Phase 1

---

## Phase 1: Foundation & Core Data Model

**Goal:** Monorepo scaffolding, database schema for core entities (tenant, organisation, position, employee), dev environment, and CI pipeline.

**Duration estimate:** 2-3 weeks

### Task 1.1: Monorepo Scaffolding

**What:** Initialize Turborepo workspace with `apps/web`, `packages/db`, `packages/api`, and `packages/shared`. Configure TypeScript project references, ESLint, Prettier, and Husky pre-commit hooks.

**Design:**

```bash
# Root package.json (pnpm workspaces)
pnpm init
pnpm add -D turbo typescript eslint prettier husky lint-staged
```

```jsonc
// turbo.json
{
  "$schema": "https://turbo.build/schema.json",
  "pipeline": {
    "build": { "dependsOn": ["^build"], "outputs": [".next/**", "dist/**"] },
    "lint": {},
    "test": { "dependsOn": ["^build"] },
    "db:migrate": { "cache": false },
    "dev": { "cache": false, "persistent": true }
  }
}
```

```typescript
// packages/shared/src/types/index.ts
export type ReadinessTier = 'ready_now' | '6_months' | '1_to_2_years' | '3_to_5_years';
export type EmploymentStatus = 'active' | 'on_leave' | 'terminated';
export type NominationStatus = 'nominated' | 'confirmed' | 'withdrawn' | 'promoted';
export type NineBoxPosition = 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9;
export type TaxonomySource = 'esco' | 'onet' | 'custom';
```

**Testing:**
- `turbo build` succeeds with zero errors across all workspaces
- `turbo lint` passes with the configured ESLint ruleset
- TypeScript project references resolve correctly (import from `@succession/shared` in `@succession/api` compiles)
- Husky pre-commit hook runs lint-staged on staged files

### Task 1.2: PostgreSQL Schema — Core Entities

**What:** Define Drizzle ORM schemas for `tenant`, `organisation_unit`, `position`, `employee`, and `skill` tables. Generate and run initial migration. Seed with a demo tenant and sample org hierarchy.

**Design:**

```typescript
// packages/db/src/schema/tenant.ts
import { pgTable, uuid, varchar, jsonb, timestamp, boolean } from 'drizzle-orm/pg-core';

export const tenant = pgTable('tenant', {
  id: uuid('id').primaryKey().defaultRandom(),
  name: varchar('name', { length: 255 }).notNull(),
  slug: varchar('slug', { length: 100 }).notNull().unique(),
  config: jsonb('config').notNull().default('{}'),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
});

// packages/db/src/schema/employee.ts
export const employee = pgTable('employee', {
  id: uuid('id').primaryKey().defaultRandom(),
  tenantId: uuid('tenant_id').notNull().references(() => tenant.id),
  externalHrisId: varchar('external_hris_id', { length: 100 }),
  hrisSource: varchar('hris_source', { length: 50 }),
  firstName: varchar('first_name', { length: 100 }).notNull(),
  lastName: varchar('last_name', { length: 100 }).notNull(),
  email: varchar('email', { length: 255 }),
  currentPositionId: uuid('current_position_id').references(() => position.id),
  organisationUnitId: uuid('organisation_unit_id').references(() => organisationUnit.id),
  hireDate: date('hire_date'),
  employmentStatus: varchar('employment_status', { length: 20 }).notNull().default('active'),
  profile: jsonb('profile').notNull().default('{}'),
  demographics: jsonb('demographics').notNull().default('{}'),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
});
```

**Testing:**
- Migration creates all 5 tables with correct column types and constraints
- `tenant.config` accepts and returns valid JSON (insert/select round-trip)
- `employee.demographics` JSONB column accepts jurisdiction-specific payloads (EU GDPR categories, US EEO categories)
- Foreign key constraint prevents inserting employee with non-existent `tenant_id`
- GIN index on `employee.profile` is created (verify via `\di` in psql)
- Seed script populates demo tenant with 3-level org hierarchy (company → 2 divisions → 4 departments), 5 positions, and 10 employees

### Task 1.3: Docker Compose Dev Environment

**What:** Configure `docker-compose.yml` with PostgreSQL 16, the API server (hot-reload), and the Next.js dev server. Include health checks and volume mounts.

**Design:**

```yaml
# deploy/docker-compose.yml
services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: succession_dev
      POSTGRES_USER: succession
      POSTGRES_PASSWORD: dev_password
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U succession"]
      interval: 5s
      timeout: 5s
      retries: 5

  api:
    build:
      context: ..
      dockerfile: deploy/Dockerfile.api
      target: development
    ports:
      - "3001:3001"
    environment:
      DATABASE_URL: postgresql://succession:dev_password@postgres:5432/succession_dev
    depends_on:
      postgres:
        condition: service_healthy
    volumes:
      - ../packages:/app/packages

  web:
    build:
      context: ..
      dockerfile: deploy/Dockerfile.web
      target: development
    ports:
      - "3000:3000"
    depends_on:
      - api

volumes:
  pgdata:
```

**Testing:**
- `docker compose up` starts all 3 services within 60 seconds
- PostgreSQL health check passes
- API server responds to `GET /health` with `200 OK`
- Next.js dev server loads at `http://localhost:3000`
- Code change in `packages/api` triggers hot-reload (verify log output)

### Task 1.4: CI Pipeline

**What:** GitHub Actions workflow for lint, type-check, unit tests, and migration verification on every PR against `main`.

**Design:**

```yaml
# .github/workflows/ci.yml
name: CI
on:
  pull_request:
    branches: [main]
  push:
    branches: [main]

jobs:
  check:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_DB: succession_test
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
        ports: ["5432:5432"]
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: pnpm
      - run: pnpm install --frozen-lockfile
      - run: pnpm turbo lint
      - run: pnpm turbo build
      - run: pnpm turbo db:migrate
        env:
          DATABASE_URL: postgresql://test:test@localhost:5432/succession_test
      - run: pnpm turbo test
        env:
          DATABASE_URL: postgresql://test:test@localhost:5432/succession_test
```

**Testing:**
- CI workflow completes green on a clean PR
- Intentional lint error causes CI failure
- Migration runs successfully against the ephemeral PostgreSQL service
- Unit tests in `packages/db` verify schema definitions compile

### Definition of Done — Phase 1
- [ ] Monorepo builds with zero TypeScript errors
- [ ] All 5 core tables exist with correct constraints and indexes
- [ ] Demo seed runs idempotently (re-run does not duplicate data)
- [ ] `docker compose up` starts a working dev environment from scratch
- [ ] CI pipeline passes on `main` branch
- [ ] OpenAPI stub route (`GET /api/v1/health`) returns `200`

---

## Phase 2: Authentication, RBAC & Audit

**Goal:** OIDC-based authentication, role-based access control (CHRO, HRBP, manager, executive, admin), tenant scoping, and audit logging.

**Duration estimate:** 2-3 weeks

### Task 2.1: OIDC Authentication with NextAuth.js v5

**What:** Configure NextAuth.js with OIDC provider support (Keycloak for self-hosted, Google/Azure AD for cloud). Implement session management, CSRF protection, and JWT-based API token issuance.

**Design:**

```typescript
// apps/web/src/app/api/auth/[...nextauth]/route.ts
import NextAuth from 'next-auth';
import KeycloakProvider from 'next-auth/providers/keycloak';
import GoogleProvider from 'next-auth/providers/google';
import { DrizzleAdapter } from '@auth/drizzle-adapter';
import { db } from '@succession/db';

export const { handlers, auth, signIn, signOut } = NextAuth({
  adapter: DrizzleAdapter(db),
  providers: [
    KeycloakProvider({
      clientId: process.env.KEYCLOAK_CLIENT_ID!,
      clientSecret: process.env.KEYCLOAK_CLIENT_SECRET!,
      issuer: process.env.KEYCLOAK_ISSUER!,
    }),
    GoogleProvider({
      clientId: process.env.GOOGLE_CLIENT_ID!,
      clientSecret: process.env.GOOGLE_CLIENT_SECRET!,
    }),
  ],
  callbacks: {
    async session({ session, user }) {
      // Attach tenant_id and roles to session
      const userAccount = await db.query.userAccount.findFirst({
        where: eq(userAccount.email, user.email),
      });
      session.user.tenantId = userAccount?.tenantId;
      session.user.roles = userAccount?.roles;
      return session;
    },
  },
});
```

**Testing:**
- OIDC login flow redirects to Keycloak, authenticates, and returns to the application with a valid session
- Session contains `tenantId` and `roles` from `user_account` table
- Unauthenticated request to any protected page redirects to `/login`
- Session expires after configured timeout (default 24 hours)
- CSRF token is validated on all state-changing requests
- API routes validate JWT bearer token and extract tenant context

### Task 2.2: Role-Based Access Control (RBAC)

**What:** Implement RBAC with five roles (admin, chro, hrbp, manager, executive) scoped optionally to `organisation_unit`. Create Fastify plugin that attaches permissions to request context and enforces route-level access.

**Design:**

```typescript
// packages/api/src/plugins/rbac.ts
import fp from 'fastify-plugin';

export const PERMISSIONS = {
  'succession:read': ['admin', 'chro', 'hrbp', 'manager', 'executive'],
  'succession:write': ['admin', 'chro', 'hrbp'],
  'nomination:create': ['admin', 'chro', 'hrbp', 'manager'],
  'nomination:approve': ['admin', 'chro', 'hrbp'],
  'calibration:facilitate': ['admin', 'chro', 'hrbp'],
  'demographics:read': ['admin', 'chro'],  // Restricted for GDPR
  'ai:review': ['admin', 'chro', 'hrbp'],
  'tenant:admin': ['admin'],
  'reports:board': ['admin', 'chro', 'executive'],
  'hris:manage': ['admin'],
} as const;

export default fp(async (fastify) => {
  fastify.decorateRequest('permissions', []);
  fastify.addHook('preHandler', async (request) => {
    const { roles, tenantId, orgUnitId } = request.user;
    request.permissions = Object.entries(PERMISSIONS)
      .filter(([, allowedRoles]) => roles.some(r => allowedRoles.includes(r.role)))
      .map(([perm]) => perm);
  });
});

// Usage in route
fastify.get('/succession-plans', {
  preHandler: [requirePermission('succession:read')],
  handler: async (request) => { /* ... */ },
});
```

**Testing:**
- Admin role has all permissions
- CHRO role can read demographics; HRBP cannot
- Manager can create nominations but cannot approve them
- Executive can read succession plans and board reports but cannot write
- Role scoped to org unit X cannot access succession plans for org unit Y
- Request to route without required permission returns `403 Forbidden`
- Role check includes tenant isolation (user in tenant A cannot access tenant B data)

### Task 2.3: Audit Logging

**What:** Implement automatic audit logging for all state-changing operations via a Fastify hook. Log entity type, entity ID, old/new values as JSONB, acting user, and IP address to the `audit_log` table.

**Design:**

```typescript
// packages/api/src/plugins/audit.ts
import fp from 'fastify-plugin';

export default fp(async (fastify) => {
  fastify.addHook('onSend', async (request, reply) => {
    if (['POST', 'PUT', 'PATCH', 'DELETE'].includes(request.method) && reply.statusCode < 400) {
      const auditEntry = request.auditContext;
      if (auditEntry) {
        await db.insert(auditLog).values({
          tenantId: request.user.tenantId,
          userId: request.user.id,
          action: auditEntry.action,
          entityType: auditEntry.entityType,
          entityId: auditEntry.entityId,
          changes: { old: auditEntry.oldValues, new: auditEntry.newValues },
          requestMetadata: {
            ip: request.ip,
            userAgent: request.headers['user-agent'],
          },
        });
      }
    }
  });
});
```

**Testing:**
- Creating a succession plan writes an audit log entry with action `succession_plan.create`
- Updating a nomination records both `old` and `new` values in JSONB
- Audit log entry includes the correct `user_id`, `tenant_id`, and IP address
- Deleting an entity records the deleted values in `old_values`
- Audit log is append-only (no UPDATE or DELETE operations succeed on the table via the application)
- Query audit log by entity type and ID returns chronological history

### Task 2.4: Tenant Scoping Middleware

**What:** Ensure every database query is automatically scoped to the authenticated user's tenant. Implement as a Drizzle query wrapper that injects `WHERE tenant_id = ?` on all queries.

**Design:**

```typescript
// packages/db/src/tenant-scope.ts
export function withTenantScope<T>(tenantId: string, queryFn: (db: DrizzleDB) => Promise<T>): Promise<T> {
  // Use Drizzle's RLS-like pattern to inject tenant filtering
  return db.transaction(async (tx) => {
    await tx.execute(sql`SET LOCAL app.current_tenant_id = ${tenantId}`);
    return queryFn(tx);
  });
}

// PostgreSQL RLS policy (applied via migration)
// CREATE POLICY tenant_isolation ON employee
//   USING (tenant_id = current_setting('app.current_tenant_id')::uuid);
```

**Testing:**
- User in tenant A cannot query employees from tenant B
- Direct SQL query without tenant context returns zero rows (RLS enforcement)
- API route that forgets tenant scoping still returns only tenant-scoped data (defense in depth)
- Cross-tenant employee count matches expected values for each tenant
- Superadmin (admin role without tenant scope) can query across tenants when explicitly requested

### Definition of Done — Phase 2
- [ ] OIDC login/logout flow works end-to-end with at least one provider
- [ ] All 5 roles are seeded and permission checks are enforced on all routes
- [ ] Audit log captures all create/update/delete operations with old/new values
- [ ] Tenant isolation verified: no cross-tenant data leakage in any API route
- [ ] Session management works (login, session refresh, forced logout)
- [ ] API returns 401 for unauthenticated and 403 for unauthorized requests

---

## Phase 3: Succession Plans, Nominations & 9-Box Grid

**Goal:** Core succession planning functionality — create succession plans for key positions, nominate successors with readiness tiers, and visualize the 9-box performance/potential grid.

**Duration estimate:** 3-4 weeks

### Task 3.1: Succession Plan CRUD API

**What:** REST API endpoints for creating, reading, updating, and archiving succession plans. A succession plan is linked to a key position and owned by an HRBP or CHRO.

**Design:**

```typescript
// packages/api/src/routes/succession/plans.ts
const successionPlanSchema = {
  body: z.object({
    positionId: z.string().uuid(),
    ownerId: z.string().uuid(),
    reviewFrequencyMonths: z.number().int().min(1).max(36).default(12),
    planConfig: z.object({
      minimumBenchDepth: z.number().int().min(1).default(3),
      requireDiversityInBench: z.boolean().default(false),
      autoAlertOnSingleSuccessor: z.boolean().default(true),
    }).optional(),
  }),
};

fastify.post('/api/v1/succession-plans', {
  schema: { body: successionPlanSchema.body },
  preHandler: [requirePermission('succession:write')],
  handler: async (request, reply) => {
    const plan = await successionService.createPlan({
      ...request.body,
      tenantId: request.user.tenantId,
    });
    request.auditContext = {
      action: 'succession_plan.create',
      entityType: 'succession_plan',
      entityId: plan.id,
      newValues: plan,
    };
    return reply.status(201).send(plan);
  },
});

// GET /api/v1/succession-plans — list with filters
// GET /api/v1/succession-plans/:id — detail with nominations
// PATCH /api/v1/succession-plans/:id — update
// POST /api/v1/succession-plans/:id/archive — archive
```

**Testing:**
- `POST /succession-plans` with valid body creates plan and returns 201
- `POST /succession-plans` with non-existent `positionId` returns 400
- `GET /succession-plans` returns only plans for the user's tenant
- `GET /succession-plans/:id` includes nested nominations array
- `PATCH /succession-plans/:id` updates `reviewFrequencyMonths` and triggers audit log
- Archiving a plan sets `status = 'archived'` and preserves all nominations
- Manager role gets 403 when attempting to create a plan (requires `succession:write`)

### Task 3.2: Successor Nomination API

**What:** Endpoints for nominating employees as successors for a plan, updating readiness tiers, ranking candidates, and withdrawing nominations.

**Design:**

```typescript
// packages/api/src/routes/succession/nominations.ts
const nominationSchema = z.object({
  employeeId: z.string().uuid(),
  readinessTier: z.string(), // validated against tenant config
  rankOrder: z.number().int().min(1).optional(),
  readinessAssessment: z.object({
    riskOfLoss: z.enum(['low', 'medium', 'high']).optional(),
    impactOfLoss: z.enum(['low', 'medium', 'high']).optional(),
    developmentNotes: z.string().optional(),
    customScores: z.record(z.number()).optional(),
  }).optional(),
});

// POST /api/v1/succession-plans/:planId/nominations
// PATCH /api/v1/succession-plans/:planId/nominations/:id
// DELETE /api/v1/succession-plans/:planId/nominations/:id (withdraw)
// PUT /api/v1/succession-plans/:planId/nominations/reorder (bulk rank update)
```

**Testing:**
- Nominating an employee for a plan creates a nomination with `status = 'nominated'`
- Cannot nominate the same employee twice for the same plan (unique constraint, returns 409)
- Readiness tier is validated against the tenant's configured tiers (reject invalid tier value)
- Reordering nominations updates `rank_order` for all nominations atomically
- Withdrawing a nomination sets `status = 'withdrawn'` (soft delete)
- Nomination includes `readinessAssessment` JSONB with custom scores matching tenant config
- Audit log records nomination creation with `nominated_by` user

### Task 3.3: Assessment & 9-Box Grid API

**What:** Endpoints for recording performance and potential assessments that populate the 9-box grid. Support individual assessment entry and batch import from HRIS.

**Design:**

```typescript
// packages/api/src/routes/assessments/index.ts
const assessmentSchema = z.object({
  employeeId: z.string().uuid(),
  assessmentPeriodStart: z.string().date(),
  assessmentPeriodEnd: z.string().date(),
  performanceRating: z.number().int().min(1).max(5),
  potentialRating: z.number().int().min(1).max(5),
  source: z.enum(['manager', 'calibration', 'ai_suggested', 'hris_import']),
  assessedBy: z.string().uuid().optional(),
  dimensions: z.record(z.unknown()).optional(),
});

// POST /api/v1/assessments — single assessment
// POST /api/v1/assessments/batch — batch import
// GET /api/v1/assessments/nine-box?orgUnitId=...&period=... — 9-box grid data
```

```typescript
// packages/api/src/services/nine-box.ts
export async function getNineBoxData(tenantId: string, filters: NineBoxFilters) {
  // Returns employees grouped by 9-box position (1-9)
  // with latest assessment per employee
  const results = await db.query.assessment.findMany({
    where: and(
      eq(assessment.tenantId, tenantId),
      eq(assessment.assessmentPeriodEnd, filters.period),
    ),
    with: { employee: true },
    orderBy: desc(assessment.createdAt),
  });

  // Group by computed 9-box position
  return groupByNineBox(results, tenantConfig.nineBoxScale);
}
```

**Testing:**
- Recording an assessment for an employee creates the record with correct ratings
- `GET /nine-box` returns employees grouped into 9 cells based on (performance, potential) ratings
- Batch import of 100 assessments succeeds within 2 seconds
- 9-box grid data respects org unit filter (only employees in selected unit)
- When two assessments exist for same period, most recent takes precedence
- Assessment `dimensions` JSONB accepts tenant-specific fields (leadership, innovation scores)
- Scale of 1-3 and 1-5 both supported depending on tenant config

### Task 3.4: 9-Box Grid UI Component

**What:** Interactive 9-box grid React component. Employees appear as cards in their grid cell. Click to view employee detail. Drag-and-drop to propose assessment changes (creates draft calibration adjustments).

**Design:**

```typescript
// apps/web/src/components/nine-box/NineBoxGrid.tsx
interface NineBoxGridProps {
  data: NineBoxData;
  labels: { performance: string[]; potential: string[] };
  onEmployeeClick: (employeeId: string) => void;
  onDragEnd?: (employeeId: string, newPosition: NineBoxPosition) => void;
  readOnly?: boolean;
}

export function NineBoxGrid({ data, labels, onEmployeeClick, onDragEnd, readOnly }: NineBoxGridProps) {
  return (
    <div className="grid grid-cols-3 grid-rows-3 gap-1 aspect-square max-w-4xl">
      {/* Y-axis: Potential (high at top) */}
      {[3, 2, 1].map((potential) =>
        [1, 2, 3].map((performance) => {
          const position = (potential - 1) * 3 + performance;
          const employees = data[position] || [];
          return (
            <NineBoxCell
              key={position}
              position={position}
              employees={employees}
              bgColor={getCellColor(performance, potential)}
              onEmployeeClick={onEmployeeClick}
              onDragEnd={onDragEnd}
              readOnly={readOnly}
            />
          );
        })
      )}
    </div>
  );
}
```

**Testing:**
- Grid renders 9 cells with correct axis labels from tenant config
- Employees appear in correct cells based on their latest assessment ratings
- Clicking an employee card opens detail side panel
- Drag-and-drop from cell (2,2) to cell (3,3) creates a draft calibration adjustment
- Read-only mode disables drag-and-drop (used for executive view)
- Empty cells display count of zero
- Grid is responsive: adapts to screen sizes down to 768px width
- Cell colors follow standard 9-box coloring (green top-right, red bottom-left)
- Grid data updates when period filter is changed

### Task 3.5: Succession Plan UI

**What:** Succession plan detail page showing the key position, its successors ranked by readiness, and actions to nominate/update/withdraw. Succession plan list page with filters.

**Design:**

```typescript
// apps/web/src/app/(dashboard)/succession/[planId]/page.tsx
// Server component that fetches plan detail with nominations
export default async function SuccessionPlanPage({ params }: { params: { planId: string } }) {
  const plan = await api.getSuccessionPlan(params.planId);
  return (
    <div className="space-y-6">
      <PlanHeader position={plan.position} owner={plan.owner} status={plan.status} />
      <BenchSummaryCard
        readyNow={plan.nominations.filter(n => n.readinessTier === 'ready_now').length}
        oneToTwo={plan.nominations.filter(n => n.readinessTier === '1_to_2_years').length}
        threeToFive={plan.nominations.filter(n => n.readinessTier === '3_to_5_years').length}
      />
      <NominationList
        nominations={plan.nominations}
        onReorder={handleReorder}
        onUpdateReadiness={handleUpdateReadiness}
        onWithdraw={handleWithdraw}
      />
      <NominateButton planId={plan.id} />
    </div>
  );
}
```

**Testing:**
- Plan detail page displays position title, owner, and status
- Nominations are listed in rank order with readiness tier badges
- "Nominate successor" button opens employee search dialog
- Drag-to-reorder nominations updates rank order via API
- Readiness tier can be changed via dropdown on each nomination card
- Withdraw action prompts confirmation before soft-deleting
- List page shows all succession plans with filters for status, org unit, and owner
- Plan with no nominations shows "No successors nominated" with call to action

### Definition of Done — Phase 3
- [ ] CRUD for succession plans, nominations, and assessments is complete with validation
- [ ] 9-box grid renders correctly with drag-and-drop for calibration
- [ ] Succession plan detail page shows ranked nominations with readiness tiers
- [ ] All mutations produce audit log entries
- [ ] RBAC enforced: managers can nominate but not approve; executives read-only
- [ ] OpenAPI spec generated from Fastify route schemas

---

## Phase 4: Talent Pools & Calibration Sessions

**Goal:** Configurable talent pools with membership management, and facilitated calibration session workflows.

**Duration estimate:** 2-3 weeks

### Task 4.1: Talent Pool CRUD API

**What:** REST endpoints for creating talent pools (leadership, role family, business unit, strategic program, custom), managing membership, and configuring auto-populate criteria.

**Design:**

```typescript
// packages/api/src/routes/talent-pools/index.ts
const talentPoolSchema = z.object({
  name: z.string().min(1).max(255),
  poolType: z.enum(['leadership', 'role_family', 'business_unit', 'strategic_program', 'custom']),
  ownerId: z.string().uuid(),
  criteria: z.object({
    autoPopulate: z.boolean().default(false),
    filters: z.object({
      nineBoxPositions: z.array(z.number().min(1).max(9)).optional(),
      minTenureYears: z.number().optional(),
      requiredSkills: z.array(z.string().uuid()).optional(),
      organisationUnits: z.array(z.string().uuid()).optional(),
    }).optional(),
    capacity: z.number().int().min(1).optional(),
  }).optional(),
});

// POST /api/v1/talent-pools
// GET /api/v1/talent-pools — list with filters
// GET /api/v1/talent-pools/:id — detail with members
// PATCH /api/v1/talent-pools/:id
// POST /api/v1/talent-pools/:id/members — add member
// DELETE /api/v1/talent-pools/:id/members/:employeeId — remove member
// POST /api/v1/talent-pools/:id/auto-populate — run auto-populate
```

**Testing:**
- Create a leadership talent pool with capacity of 10
- Add 5 employees to the pool with readiness tiers
- Auto-populate based on 9-box criteria (positions 6, 8, 9) returns matching employees
- Cannot add the same employee twice (unique constraint, 409)
- Removing a member removes only the membership, not the employee
- Pool detail includes member list with employee names, readiness tiers, and notes
- Pool type filter returns only pools of that type
- Auto-populate respects capacity limit

### Task 4.2: Calibration Session Workflow

**What:** Full calibration session lifecycle: plan a session, add participants (facilitator, reviewers, observers), run the session (view 9-box grid with in-session adjustment capability), record calibration decisions, and complete/finalize the session.

**Design:**

```typescript
// packages/api/src/routes/calibration/index.ts
// POST /api/v1/calibration-sessions — create session
// POST /api/v1/calibration-sessions/:id/start — transition to in_progress
// POST /api/v1/calibration-sessions/:id/decisions — record adjustment
// POST /api/v1/calibration-sessions/:id/complete — finalize
// GET /api/v1/calibration-sessions/:id — detail with decisions

const calibrationDecisionSchema = z.object({
  employeeId: z.string().uuid(),
  originalPerformance: z.number().int().min(1).max(5),
  adjustedPerformance: z.number().int().min(1).max(5),
  originalPotential: z.number().int().min(1).max(5),
  adjustedPotential: z.number().int().min(1).max(5),
  rationale: z.string().min(10), // require explanation for any adjustment
  consensusReached: z.boolean(),
});
```

```typescript
// apps/web/src/app/(dashboard)/calibration/[sessionId]/page.tsx
// Calibration session page with live 9-box grid
// - Facilitator can drag employees between cells
// - Each drag creates a calibration decision with required rationale
// - Real-time participant list shows who is viewing
// - "Complete Session" button finalizes and creates new assessment records
```

**Testing:**
- Session lifecycle transitions: planned → in_progress → completed
- Cannot add decisions to a completed session (returns 400)
- Calibration decision records original and adjusted ratings with rationale
- Completing a session creates new `assessment` records with `source = 'calibration'`
- Only facilitators can start and complete sessions (RBAC enforcement)
- Session data JSONB stores full participant list and decision history
- Decision without rationale is rejected (validation error)
- Session detail page shows before/after 9-box for each adjusted employee

### Task 4.3: Talent Pool & Calibration UI

**What:** Talent pool management page with member list, add/remove actions, and pool configuration. Calibration session facilitation UI with interactive 9-box grid.

**Design:**

The talent pool page renders pool metadata, configurable criteria editor, and a member data table with inline readiness tier editing. The calibration page embeds the NineBoxGrid component (from Task 3.4) with drag-and-drop enabled, plus a decision log panel that records each adjustment with a mandatory rationale text input.

**Testing:**
- Pool management page lists all pools with member counts
- Adding a member via employee search dialog immediately updates the member list
- Calibration session page shows the 9-box grid populated with employees from the selected org unit
- Dragging an employee between cells prompts for rationale text before recording the decision
- Decision log panel shows chronological list of adjustments made during the session
- "Complete Session" button is disabled until at least one decision is recorded
- Completed session shows read-only view of all adjustments

### Definition of Done — Phase 4
- [ ] Talent pools CRUD complete with auto-populate functionality
- [ ] Calibration session lifecycle (plan → start → decide → complete) works end-to-end
- [ ] Calibration decisions create new assessment records on completion
- [ ] Talent pool and calibration UIs are functional with appropriate RBAC
- [ ] All mutations audited

---

## Phase 5: HRIS Integration Layer

**Goal:** Pluggable HRIS connector architecture with initial support for BambooHR (API key auth), ADP (OAuth 2.0), and Finch (unified API). Scheduled and on-demand sync of employee, position, and performance data.

**Duration estimate:** 3-4 weeks

### Task 5.1: HRIS Connector Architecture

**What:** Design a pluggable connector interface that abstracts HRIS-specific API details behind a common contract. Each connector implements the interface for a specific provider.

**Design:**

```typescript
// packages/api/src/services/hris/connector.interface.ts
export interface HrisConnector {
  readonly provider: string;

  testConnection(): Promise<{ ok: boolean; message: string }>;

  fetchEmployees(since?: Date): Promise<HrisEmployee[]>;
  fetchPositions(): Promise<HrisPosition[]>;
  fetchPerformanceRatings(periodEnd: Date): Promise<HrisAssessment[]>;

  mapToEmployee(hrisEmployee: HrisEmployee): Partial<InsertEmployee>;
  mapToPosition(hrisPosition: HrisPosition): Partial<InsertPosition>;
  mapToAssessment(hrisAssessment: HrisAssessment): Partial<InsertAssessment>;
}

// Normalized HRIS data types
export interface HrisEmployee {
  externalId: string;
  firstName: string;
  lastName: string;
  email: string;
  positionCode?: string;
  departmentName?: string;
  hireDate?: string;
  status: string;
  customFields?: Record<string, unknown>;
}

// packages/api/src/services/hris/bamboohr.connector.ts
export class BambooHRConnector implements HrisConnector {
  readonly provider = 'bamboohr';

  constructor(private config: { subdomain: string; apiKey: string; fieldMappings?: Record<string, string> }) {}

  async fetchEmployees(since?: Date): Promise<HrisEmployee[]> {
    const response = await fetch(
      `https://api.bamboohr.com/api/gateway.php/${this.config.subdomain}/v1/employees/directory`,
      { headers: { Authorization: `Basic ${btoa(this.config.apiKey + ':x')}`, Accept: 'application/json' } }
    );
    const data = await response.json();
    return data.employees.map(this.normalizeEmployee);
  }
  // ...
}
```

**Testing:**
- `BambooHRConnector.testConnection()` returns `{ ok: true }` with valid credentials and `{ ok: false }` with invalid
- `fetchEmployees()` returns normalized `HrisEmployee[]` regardless of BambooHR's custom field configuration
- `mapToEmployee()` correctly maps BambooHR fields to internal employee schema
- Connector factory creates correct connector for 'bamboohr', 'adp', 'finch' providers
- Unknown provider throws descriptive error

### Task 5.2: HRIS Sync Engine

**What:** Background sync engine that runs connectors on a configurable schedule, performs upsert (insert or update based on `external_hris_id`), records sync logs, and handles partial failures.

**Design:**

```typescript
// packages/api/src/services/hris/sync-engine.ts
export class HrisSyncEngine {
  async runSync(connectionId: string): Promise<SyncResult> {
    const connection = await db.query.hrisConnection.findFirst({
      where: eq(hrisConnection.id, connectionId),
    });

    const connector = ConnectorFactory.create(connection.provider, connection.config);
    const syncLog = await this.createSyncLog(connectionId);

    try {
      // Fetch and upsert employees
      const hrisEmployees = await connector.fetchEmployees(connection.lastSyncAt);
      let synced = 0, failed = 0;
      for (const hrisEmp of hrisEmployees) {
        try {
          await db.insert(employee)
            .values({ ...connector.mapToEmployee(hrisEmp), tenantId: connection.tenantId })
            .onConflictDoUpdate({
              target: [employee.tenantId, employee.hrisSource, employee.externalHrisId],
              set: connector.mapToEmployee(hrisEmp),
            });
          synced++;
        } catch (e) {
          failed++;
          // Log individual record failure
        }
      }

      await this.completeSyncLog(syncLog.id, 'success', synced, failed);
      return { synced, failed };
    } catch (e) {
      await this.completeSyncLog(syncLog.id, 'failed', 0, 0, e.message);
      throw e;
    }
  }
}
```

**Testing:**
- Sync creates new employees for first-time sync
- Re-sync updates existing employees matched by `(tenant_id, hris_source, external_hris_id)` composite key
- Sync log records `started_at`, `completed_at`, `records_synced`, `records_failed`
- Partial failure (1 of 50 records invalid) records `status = 'partial'` with 49 synced, 1 failed
- Complete failure (API unreachable) records `status = 'failed'` with error details
- Sync does not delete employees that no longer appear in HRIS (soft handling — sets `employment_status = 'terminated'` only if confirmed)
- Concurrent sync attempts for the same connection are blocked (distributed lock)

### Task 5.3: HRIS Connection Management UI

**What:** Settings page for managing HRIS connections: add connection (select provider, enter credentials), test connection, trigger manual sync, and view sync history.

**Design:**

The settings page renders a list of configured HRIS connections with status badges (connected/disconnected/error). An "Add Connection" dialog presents provider selection (BambooHR, ADP, Finch), credential input fields (with appropriate auth flow per provider), and a field mapping configuration step. Each connection card shows last sync timestamp, records synced, and a "Sync Now" button.

**Testing:**
- Adding a BambooHR connection with valid API key shows "Connected" status after test
- "Sync Now" button triggers sync and shows progress indicator
- Sync history shows each sync attempt with timestamp, status, and record counts
- Invalid credentials show clear error message
- Field mapping configuration allows mapping HRIS custom fields to succession tool fields
- Only admin role can access HRIS settings (RBAC enforcement)

### Definition of Done — Phase 5
- [ ] BambooHR connector syncs employees and positions
- [ ] ADP connector authenticates via OAuth 2.0 and syncs employee data
- [ ] Finch connector provides unified access to 250+ HRIS systems
- [ ] Sync engine runs on schedule and handles partial failures
- [ ] Sync logs provide troubleshooting data
- [ ] HRIS connection management UI complete with test and manual sync

---

## Phase 6: Bench Strength Dashboard & Risk Reporting

**Goal:** Visual bench strength dashboard showing coverage heat maps, single-point risk alerts, and risk concentration reporting.

**Duration estimate:** 2-3 weeks

### Task 6.1: Bench Strength API

**What:** API endpoint that computes bench strength metrics for each key position: total successors, ready-now count, single-point risk flag, and composite risk score.

**Design:**

```typescript
// packages/api/src/services/bench-strength.ts
export interface BenchStrengthEntry {
  positionId: string;
  positionTitle: string;
  level: string;
  orgUnitName: string;
  incumbentName: string | null;
  totalSuccessors: number;
  readyNowCount: number;
  ready1To2Count: number;
  ready3To5Count: number;
  hasSinglePointRisk: boolean;  // true if totalSuccessors <= 1
  hasNoSuccessor: boolean;       // true if totalSuccessors === 0
  riskScore: number;             // 0-100 composite score
}

export async function getBenchStrength(tenantId: string, filters: BenchFilters): Promise<BenchStrengthEntry[]> {
  const results = await db.execute(sql`
    SELECT
      p.id AS position_id,
      p.title AS position_title,
      p.level,
      ou.name AS org_unit_name,
      CONCAT(e.first_name, ' ', e.last_name) AS incumbent_name,
      COUNT(sn.id) AS total_successors,
      COUNT(sn.id) FILTER (WHERE sn.readiness_tier = 'ready_now') AS ready_now_count,
      COUNT(sn.id) FILTER (WHERE sn.readiness_tier = '1_to_2_years') AS ready_1_to_2_count,
      COUNT(sn.id) FILTER (WHERE sn.readiness_tier = '3_to_5_years') AS ready_3_to_5_count
    FROM position p
    LEFT JOIN succession_plan sp ON sp.position_id = p.id AND sp.status = 'active'
    LEFT JOIN successor_nomination sn ON sn.succession_plan_id = sp.id AND sn.nomination_status != 'withdrawn'
    LEFT JOIN organisation_unit ou ON ou.id = p.organisation_unit_id
    LEFT JOIN employee e ON e.current_position_id = p.id
    WHERE p.tenant_id = ${tenantId}
      AND p.is_key_position = true
    GROUP BY p.id, p.title, p.level, ou.name, e.first_name, e.last_name
    ORDER BY total_successors ASC, p.level DESC
  `);

  return results.map(computeRiskScore);
}
```

**Testing:**
- Position with 0 successors has `hasNoSuccessor = true` and risk score of 100
- Position with 1 successor has `hasSinglePointRisk = true` and risk score of 75+
- Position with 3+ successors including a ready-now has risk score below 25
- Filter by org unit returns only positions within that unit
- Filter by level (e.g., 'C-Suite') returns only C-level positions
- Bench strength data matches hand-calculated values from test seed data

### Task 6.2: Bench Strength Heat Map UI

**What:** Dashboard page with a heat map visualization. Rows are org units (or levels), columns are risk categories. Color coding: red (no successor), orange (single point risk), yellow (inadequate depth), green (healthy bench). Clicking a cell drills into the positions in that category.

**Design:**

```typescript
// apps/web/src/components/heat-map/BenchStrengthHeatMap.tsx
interface HeatMapProps {
  data: BenchStrengthEntry[];
  groupBy: 'orgUnit' | 'level';
  onCellClick: (groupKey: string, riskCategory: string) => void;
}

// Risk categories for columns:
// "No Successor" | "Single Point Risk" | "Shallow Bench (2)" | "Healthy (3+)"
// Color mapping:
// No Successor → bg-red-600
// Single Point Risk → bg-orange-500
// Shallow Bench → bg-yellow-400
// Healthy → bg-green-500
// Cell content: count of positions in that category
```

**Testing:**
- Heat map renders with correct color coding for each risk category
- Clicking a red cell navigates to list of positions with no successor
- Group by org unit shows department-level aggregation
- Group by level shows C-Suite, VP, Director, Manager rows
- Heat map totals match the bench strength API response
- Dashboard loads in under 2 seconds for 200 key positions

### Task 6.3: Risk Alerts

**What:** Automated alerts when bench strength drops below configurable thresholds. Alerts appear in the UI notification center and optionally trigger email notifications.

**Design:**

```typescript
// packages/api/src/services/risk-alerts.ts
export async function evaluateRiskAlerts(tenantId: string): Promise<Alert[]> {
  const benchData = await getBenchStrength(tenantId, {});
  const alerts: Alert[] = [];

  for (const entry of benchData) {
    if (entry.hasNoSuccessor) {
      alerts.push({
        severity: 'critical',
        type: 'no_successor',
        message: `${entry.positionTitle} has no designated successor`,
        positionId: entry.positionId,
      });
    } else if (entry.hasSinglePointRisk) {
      alerts.push({
        severity: 'warning',
        type: 'single_point_risk',
        message: `${entry.positionTitle} has only one successor`,
        positionId: entry.positionId,
      });
    }
  }
  return alerts;
}
```

**Testing:**
- Position with no successor generates a critical alert
- Position with single successor generates a warning alert
- Position with 3 successors generates no alert
- Alerts are tenant-scoped
- Alert count badge appears in the UI navigation bar
- Alert links to the corresponding succession plan
- Alert evaluation runs after HRIS sync completes (employee departure could change bench strength)

### Definition of Done — Phase 6
- [ ] Bench strength API returns correct metrics for all key positions
- [ ] Heat map visualization renders with correct risk color coding
- [ ] Drill-down from heat map cell to position list works
- [ ] Risk alerts generated for no-successor and single-point-risk positions
- [ ] Dashboard loads within performance budget (< 2s for 200 positions)

---

## Phase 7: Skills Taxonomy & Graph Layer

**Goal:** Load ESCO and O*NET skills taxonomies, implement the graph layer (graph_node, graph_edge), and build skills adjacency matching infrastructure.

**Duration estimate:** 3-4 weeks

### Task 7.1: ESCO Taxonomy Loader

**What:** Script that fetches the ESCO classification from the European Commission API and loads skills, occupations, and their relationships into the `skill` table and graph layer.

**Design:**

```typescript
// packages/db/src/seed/esco-skills.ts
export async function loadEscoTaxonomy(): Promise<void> {
  // ESCO API: https://esco.ec.europa.eu/en/use-esco/use-esco-services-api
  // Load ~13,939 skills with:
  // - broaderSkill / narrowerSkill hierarchy edges
  // - relatedSkill adjacency edges
  // - skill → occupation edges
  // Rate limit: respect ESCO API rate limits (~100 req/min)

  const skills = await fetchEscoSkills(); // paginated fetch
  for (const skill of skills) {
    // Insert into relational skill table
    const skillRow = await db.insert(skillTable).values({
      name: skill.preferredLabel.en,
      taxonomySource: 'esco',
      externalUri: skill.uri,
      skillType: skill.skillType, // 'skill' | 'competence' | 'knowledge'
      metadata: { reusabilityLevel: skill.reusabilityLevel, languages: skill.preferredLabel },
    }).returning();

    // Insert graph node
    await db.insert(graphNode).values({
      nodeType: 'skill',
      entityId: skillRow.id,
      label: skill.preferredLabel.en,
      properties: { escoUri: skill.uri, taxonomy: 'esco', reusability: skill.reusabilityLevel },
    });
  }

  // Load edges (broaderSkill, narrowerSkill, relatedSkill)
  for (const relation of await fetchEscoRelations()) {
    const sourceNode = await findGraphNode('skill', relation.sourceUri);
    const targetNode = await findGraphNode('skill', relation.targetUri);
    await db.insert(graphEdge).values({
      sourceNodeId: sourceNode.id,
      targetNodeId: targetNode.id,
      edgeType: relation.type, // 'broader_skill', 'related_skill'
      weight: relation.type === 'related_skill' ? 0.7 : 1.0,
    });
  }
}
```

**Testing:**
- Loader creates 13,000+ skill records in the `skill` table
- Each skill has a corresponding `graph_node` record
- `broader_skill` edges form a valid hierarchy (no cycles)
- `related_skill` edges connect skills across categories
- Loader is idempotent (re-run does not duplicate skills, uses upsert on `external_uri`)
- Skill search by name returns matching ESCO skills
- Graph traversal from a skill returns its related skills within N hops

### Task 7.2: O*NET Taxonomy Loader

**What:** Script that loads O*NET occupational data and skills via the O*NET Web Services API, creating skills and occupation records in the graph layer.

**Design:**

Similar structure to Task 7.1, but consuming the O*NET API (`https://services.onetcenter.org/`) with occupation codes (SOC format) and skills mapped via `taxonomy_source = 'onet'`. The `skill` table supports both ESCO and O*NET entries. Cross-taxonomy edges (ESCO skill ↔ O*NET equivalent) are created based on label matching or published crosswalks.

**Testing:**
- Loader creates 900+ occupation records and associated skills
- O*NET skills have `taxonomy_source = 'onet'` and valid `external_uri`
- Cross-taxonomy edges link ESCO and O*NET skills with matching labels
- Occupation → skill edges include importance and level weights from O*NET

### Task 7.3: Graph Layer Schema & Sync

**What:** Drizzle schema for `graph_node` and `graph_edge` tables. Sync process that creates/updates graph nodes when relational entities change (employee, position, skill changes trigger graph updates).

**Design:**

```typescript
// packages/db/src/schema/graph.ts
export const graphNode = pgTable('graph_node', {
  id: uuid('id').primaryKey().defaultRandom(),
  nodeType: varchar('node_type', { length: 50 }).notNull(),
  entityId: uuid('entity_id').notNull(),
  label: varchar('label', { length: 255 }).notNull(),
  properties: jsonb('properties').notNull().default('{}'),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
});

export const graphEdge = pgTable('graph_edge', {
  id: uuid('id').primaryKey().defaultRandom(),
  sourceNodeId: uuid('source_node_id').notNull().references(() => graphNode.id, { onDelete: 'cascade' }),
  targetNodeId: uuid('target_node_id').notNull().references(() => graphNode.id, { onDelete: 'cascade' }),
  edgeType: varchar('edge_type', { length: 50 }).notNull(),
  weight: numeric('weight', { precision: 5, scale: 2 }).default('1.0'),
  properties: jsonb('properties').notNull().default('{}'),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
});

// Sync: when an employee_skill record is created, create a has_skill edge
// When a position_skill requirement is set, create a requires_skill edge
// When a nomination is created, create a nominated_for edge
```

**Testing:**
- Creating an `employee_skill` record triggers creation of a `has_skill` graph edge
- Deleting an `employee_skill` record removes the corresponding graph edge
- Graph node for an employee has `properties` containing tenure, 9-box position, and flight risk
- Graph edges have correct `weight` values (proficiency for has_skill, importance for requires_skill)
- `UNIQUE (node_type, entity_id)` constraint prevents duplicate graph nodes
- GIN index on `properties` is created for efficient JSONB queries

### Task 7.4: Skills Adjacency Query Service

**What:** Service that finds employees within N skill hops of a target position's requirements. This is the foundation for AI successor identification.

**Design:**

```typescript
// packages/api/src/services/skills-adjacency.ts
export async function findCandidatesBySkillsAdjacency(
  positionId: string,
  maxHops: number = 2,
  minCoverageRatio: number = 0.5,
): Promise<SkillsAdjacencyResult[]> {
  // Uses recursive CTE to traverse related_skill edges up to maxHops
  // Returns employees ranked by adjacency score (direct match = 1.0, 1 hop = 0.7, 2 hops = 0.4)
  const results = await db.execute(sql`
    WITH target_skills AS (
      SELECT ge.target_node_id AS skill_node_id
      FROM graph_node gn
      JOIN graph_edge ge ON ge.source_node_id = gn.id AND ge.edge_type = 'requires_skill'
      WHERE gn.node_type = 'position' AND gn.entity_id = ${positionId}
    ),
    -- ... recursive CTE for adjacent skills (see data model suggestion 4)
    -- ... candidate scoring
    SELECT employee_id, employee_name, matching_skills, adjacency_score, coverage_ratio
    FROM candidate_scores
    WHERE coverage_ratio >= ${minCoverageRatio}
    ORDER BY adjacency_score DESC
    LIMIT 50
  `);
  return results;
}
```

**Testing:**
- Employee with direct skill match scores 1.0 adjacency
- Employee with 1-hop adjacent skill scores ~0.7
- Employee with 2-hop adjacent skill scores ~0.4
- Employee with no matching or adjacent skills is excluded
- `minCoverageRatio = 0.5` filters employees with less than 50% skill coverage
- Query completes within 500ms for a graph of 10,000 skill nodes and 1,000 employees
- Results are ordered by adjacency score descending

### Definition of Done — Phase 7
- [ ] ESCO taxonomy loaded with 13,000+ skills and relationship edges
- [ ] O*NET taxonomy loaded with occupations and skills
- [ ] Graph layer syncs with relational data (employee skills, position requirements)
- [ ] Skills adjacency query returns ranked candidates within N hops
- [ ] Graph traversal performance verified for target dataset sizes
- [ ] Pluggable taxonomy: tenant config selects ESCO, O*NET, or both

---

## Phase 8: AI Successor Identification

**Goal:** Python AI microservice that generates skills-based readiness scores and successor recommendations using embeddings and the graph layer. GDPR Article 22-compliant human review workflow.

**Duration estimate:** 4-5 weeks

### Task 8.1: AI Service Scaffolding

**What:** Set up the Python FastAPI microservice with project structure, configuration, health checks, and OpenAPI documentation.

**Design:**

```python
# apps/ai-service/src/api/main.py
from fastapi import FastAPI
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    database_url: str
    api_base_url: str  # Node.js API base URL
    model_version: str = "v1.0"
    esco_embeddings_path: str = "./data/esco_embeddings.npy"

app = FastAPI(title="Succession AI Service", version="1.0.0")

@app.get("/health")
async def health():
    return {"status": "ok", "model_version": settings.model_version}

@app.post("/api/v1/recommendations/generate")
async def generate_recommendations(request: RecommendationRequest) -> RecommendationResponse:
    """Generate AI successor recommendations for a position."""
    pass

@app.post("/api/v1/readiness/score")
async def score_readiness(request: ReadinessRequest) -> ReadinessResponse:
    """Compute continuous readiness score for an employee against a position."""
    pass
```

**Testing:**
- `GET /health` returns 200 with model version
- OpenAPI spec is generated at `/docs`
- Service connects to PostgreSQL and reads employee data
- Docker container starts with correct environment variables
- Service logs are structured JSON

### Task 8.2: Skills Embedding Engine

**What:** Generate vector embeddings for ESCO/O*NET skills using sentence-transformers. Pre-compute embeddings for the full taxonomy. Use cosine similarity for skills matching that supplements the graph-based adjacency approach.

**Design:**

```python
# apps/ai-service/src/embeddings/skills_embedder.py
from sentence_transformers import SentenceTransformer
import numpy as np

class SkillsEmbedder:
    def __init__(self, model_name: str = "all-MiniLM-L6-v2"):
        self.model = SentenceTransformer(model_name)

    def embed_skills(self, skills: list[dict]) -> np.ndarray:
        """Generate embeddings for skill names and descriptions."""
        texts = [f"{s['name']}: {s.get('description', '')}" for s in skills]
        return self.model.encode(texts, normalize_embeddings=True)

    def compute_similarity(self, employee_skills: np.ndarray, position_skills: np.ndarray) -> float:
        """Compute aggregate cosine similarity between employee and position skill sets."""
        # Average pooling of employee skills vs. each position skill
        similarity_matrix = np.dot(employee_skills, position_skills.T)
        return float(np.mean(np.max(similarity_matrix, axis=0)))
```

**Testing:**
- Embedding generation for 13,000 ESCO skills completes in under 10 minutes
- Embeddings are 384-dimensional (MiniLM-L6-v2 output dimension)
- Cosine similarity between "Financial Planning" and "Budget Management" > 0.7
- Cosine similarity between "Financial Planning" and "Carpentry" < 0.3
- Pre-computed embeddings are persisted to disk and loaded on service startup
- Incremental embedding for new custom skills works without reprocessing the full taxonomy

### Task 8.3: Successor Matching Algorithm

**What:** Combine graph-based adjacency scores, embedding-based similarity scores, and readiness signals (tenure, performance ratings, development plan progress) into a composite successor fit score.

**Design:**

```python
# apps/ai-service/src/matching/successor_matcher.py
@dataclass
class SuccessorScore:
    employee_id: str
    skills_match_score: float       # from graph adjacency + embedding similarity
    readiness_score: float          # from tenure, performance, development signals
    trajectory_score: float         # from career progression rate
    overall_fit_score: float        # weighted combination
    explanation: str                # human-readable explanation
    skill_gaps: list[SkillGap]      # specific gaps identified

class SuccessorMatcher:
    WEIGHTS = {
        'skills_match': 0.40,
        'readiness': 0.35,
        'trajectory': 0.25,
    }

    async def find_successors(self, position_id: str, tenant_id: str, top_n: int = 20) -> list[SuccessorScore]:
        # 1. Get position skill requirements from graph
        # 2. Get all active employees in tenant
        # 3. For each employee:
        #    a. Compute skills_match_score (graph adjacency + embedding similarity)
        #    b. Compute readiness_score (tenure, latest assessment, dev plan progress)
        #    c. Compute trajectory_score (career progression rate, learning velocity)
        # 4. Compute overall_fit_score as weighted sum
        # 5. Generate explanation text for each candidate
        # 6. Return top N candidates sorted by overall_fit_score
        pass

    def _generate_explanation(self, scores: SuccessorScore, employee: Employee, position: Position) -> str:
        """Generate GDPR Article 22-compliant explanation of why this candidate was recommended."""
        return (
            f"Recommended based on {scores.skills_match_score:.0%} skills alignment "
            f"({len([g for g in scores.skill_gaps if g.gap <= 1])} of {total_skills} skills at or near required level), "
            f"{scores.readiness_score:.0%} readiness indicators "
            f"(tenure: {employee.tenure_years:.1f}y, latest performance: {employee.latest_rating}/5), "
            f"and {scores.trajectory_score:.0%} career trajectory score."
        )
```

**Testing:**
- Employee with 90% direct skill match and 3+ years tenure scores > 0.85 overall
- Employee with 50% skill match (all via adjacency) scores between 0.5-0.7
- Employee with high performance but poor skills match scores below 0.6 (skills weighted highest)
- Explanation text is human-readable and includes specific scores and reasoning
- Skill gaps are correctly identified (skills required by position but not held by employee)
- Algorithm processes 1,000 employees against a position within 5 seconds
- Results are deterministic (same input produces same scores)

### Task 8.4: GDPR Article 22 Human Review Workflow

**What:** API endpoints and UI for reviewing AI recommendations. No recommendation can be actioned (used in a nomination) without documented human review with reviewer identity and notes.

**Design:**

```typescript
// packages/api/src/routes/ai/recommendations.ts
// POST /api/v1/ai/recommendations/generate — trigger AI recommendation for a position
// GET /api/v1/ai/recommendations?positionId=...&status=pending — list pending reviews
// POST /api/v1/ai/recommendations/:id/review — approve/reject with notes

const reviewSchema = z.object({
  decision: z.enum(['approved', 'rejected', 'override']),
  notes: z.string().min(10, 'Review notes must explain the decision'),
});

// Enforcement: when creating a nomination from an AI recommendation,
// verify that ai_recommendation.human_review_status === 'approved'
```

```typescript
// apps/web/src/app/(dashboard)/ai/review/page.tsx
// AI Review Queue page:
// - Lists pending AI recommendations with scores and explanations
// - Each card shows: employee name, position, overall score, skill gaps
// - "Approve" and "Reject" buttons with mandatory notes field
// - "Add to Succession Plan" button only enabled after approval
// - Audit trail shows who approved and when
```

**Testing:**
- Generating recommendations creates records with `human_review_status = 'pending'`
- Approving a recommendation sets `reviewer_id`, `reviewed_at`, and `notes`
- Rejecting a recommendation records the decision and rationale
- Attempting to create a nomination from a pending/rejected recommendation returns 400
- Nomination created from an approved recommendation links back to the `ai_recommendation` record
- Review queue shows only pending recommendations for the user's org scope
- Review notes field rejects empty or too-short input (minimum 10 characters)
- Audit log records the review action with full context

### Definition of Done — Phase 8
- [ ] AI service generates successor recommendations using skills graph + embeddings
- [ ] Overall fit score combines skills match, readiness, and trajectory
- [ ] Explanations are human-readable and include specific evidence
- [ ] Human review workflow prevents unreviewed AI recommendations from becoming nominations
- [ ] Algorithm performance meets latency targets (< 5s for 1,000 employees)
- [ ] GDPR Article 22 compliance: every actioned recommendation has documented human review

---

## Phase 9: Equity Lens & Diversity Reporting

**Goal:** Configurable diversity and equity lens on succession pools, representation gap reporting, and bias detection in succession decisions.

**Duration estimate:** 2-3 weeks

### Task 9.1: Diversity Dimension Configuration

**What:** Tenant-level configuration of which diversity dimensions to track (gender, ethnicity, disability status, etc.) and the jurisdiction-appropriate categories for each. Categories vary by jurisdiction (GDPR-region vs. US EEO vs. Australian categories).

**Design:**

```typescript
// Tenant config structure (in tenant.config JSONB):
// {
//   "diversity_dimensions": ["gender", "ethnicity"],
//   "diversity_categories": {
//     "gender": ["male", "female", "non_binary", "prefer_not_to_say"],
//     "ethnicity": ["...jurisdiction-specific..."]
//   }
// }

// packages/api/src/routes/equity/config.ts
// GET /api/v1/equity/dimensions — returns configured dimensions and categories
// PUT /api/v1/equity/dimensions — update dimensions (admin only)
```

**Testing:**
- Default US tenant has EEO-1 ethnicity categories
- Default EU tenant has GDPR-minimal categories
- Custom dimensions can be added by admin
- Dimension configuration is tenant-specific (tenant A and B can have different categories)
- API rejects unknown dimensions not in tenant config

### Task 9.2: Representation Gap Analysis

**What:** For each talent pool and succession pipeline stage (identified → nominated → ready now → promoted), compute representation percentages by diversity dimension and identify gaps compared to workforce composition.

**Design:**

```typescript
// packages/api/src/services/equity.ts
export interface RepresentationGap {
  dimension: string;
  category: string;
  workforcePercent: number;
  talentPoolPercent: number;
  readyNowPercent: number;
  promotedPercent: number;
  gapFromWorkforce: number; // negative = underrepresented
}

// GET /api/v1/equity/representation?poolId=...&dimension=gender
// GET /api/v1/equity/pipeline?orgUnitId=...&dimension=ethnicity
```

**Testing:**
- If workforce is 50% female but talent pool is 30% female, `gapFromWorkforce = -20`
- Pipeline report shows representation at each stage (identified → nominated → ready now)
- Representation drops at each stage are flagged as potential bias indicators
- Only users with `demographics:read` permission can access equity reports (CHRO, admin)
- Employees who have not declared demographics are excluded from calculations (not counted as a category)
- Report handles edge cases: zero employees in a category, single-person pools

### Task 9.3: Equity Dashboard UI

**What:** Dashboard page with bar charts showing representation by dimension at each pipeline stage. Visual indication of gaps (red for underrepresentation > 10%, yellow for 5-10%).

**Design:**

The dashboard uses stacked bar charts (via recharts or similar) with one bar per pipeline stage. Color-coded badges highlight significant gaps. A drill-down table shows individual employees at each stage. Access restricted to CHRO and admin roles.

**Testing:**
- Bar chart renders correct percentages for each category at each pipeline stage
- Gap indicators appear for categories with > 5% underrepresentation
- Filter by org unit updates the chart
- Only CHRO and admin roles can access the page (RBAC)
- Empty dimensions (no employees have declared) show a "no data" message
- Export to CSV includes all representation data

### Definition of Done — Phase 9
- [ ] Diversity dimensions configurable per tenant with jurisdiction-appropriate categories
- [ ] Representation gap analysis computes gaps across succession pipeline stages
- [ ] Equity dashboard visualizes representation with gap indicators
- [ ] RBAC restricts demographics access to CHRO and admin
- [ ] GDPR compliance: demographics are self-reported, consent-tracked, and access-controlled

---

## Phase 10: Board-Ready Reports & ISO 30433 Metrics

**Goal:** Auto-generated succession risk reports for board presentation and ISO 30433/30414 metrics computation and export.

**Duration estimate:** 3-4 weeks

### Task 10.1: ISO 30433 Metrics Engine

**What:** Periodic computation of ISO/TS 30433 succession planning metrics: succession effectiveness rate, successor coverage rate, succession depth (by readiness tier), and time to fill from pipeline.

**Design:**

```typescript
// packages/api/src/services/iso-metrics.ts
export async function computeIso30433Metrics(
  tenantId: string,
  periodStart: Date,
  periodEnd: Date,
  orgUnitId?: string,
): Promise<SuccessionMetrics> {
  const totalKeyPositions = await countKeyPositions(tenantId, orgUnitId);
  const positionsWithSuccessors = await countPositionsWithSuccessors(tenantId, orgUnitId);
  const readyNowDepth = await avgReadyNowDepth(tenantId, orgUnitId);
  const effectivenessRate = await computeEffectivenessRate(tenantId, periodStart, periodEnd);

  return {
    successorCoverageRate: positionsWithSuccessors / totalKeyPositions,
    successionEffectivenessRate: effectivenessRate,
    successionDepthReadyNow: readyNowDepth,
    successionDepth1To3Years: await avgDepth(tenantId, orgUnitId, '1_to_2_years'),
    successionDepth4To5Years: await avgDepth(tenantId, orgUnitId, '3_to_5_years'),
    totalKeyPositions,
    positionsWithNoSuccessor: totalKeyPositions - positionsWithSuccessors,
  };
}

// Cron: compute and store metrics on the 1st of each month
// GET /api/v1/reports/iso-metrics?period=2026-Q1&orgUnitId=...
// GET /api/v1/reports/iso-metrics/export?format=csv|json
```

**Testing:**
- Coverage rate = (positions with >= 1 successor) / (total key positions)
- Effectiveness rate = (internal promotions from succession pool) / (total leadership appointments in period)
- Depth metrics are averages per key position
- Historical metrics are stored and queryable for trend analysis
- CSV export includes all ISO 30433 fields with headers matching ISO terminology
- JSON export follows HR Open Standards field naming
- Metrics computation completes within 10 seconds for 500 key positions

### Task 10.2: Board Report Generator

**What:** Auto-generated board-ready succession risk report combining bench strength heat map, risk concentration summary, critical position alerts, and ISO metrics trends. Exportable to PDF and PowerPoint.

**Design:**

```typescript
// packages/api/src/services/board-report.ts
export async function generateBoardReport(tenantId: string, periodEnd: Date): Promise<BoardReport> {
  return {
    executiveSummary: await generateNarrativeSummary(tenantId, periodEnd),
    benchStrengthHeatMap: await getBenchStrength(tenantId, {}),
    criticalPositions: await getCriticalPositions(tenantId), // positions with 0 successors
    riskConcentration: await getRiskConcentration(tenantId),  // org units with highest risk
    isoMetrics: await computeIso30433Metrics(tenantId, ...),
    isoMetricsTrend: await getMetricsTrend(tenantId, 4),     // last 4 periods
    equitySummary: await getEquitySummary(tenantId),
  };
}

// GET /api/v1/reports/board?period=2026-Q1
// GET /api/v1/reports/board/export?format=pdf|pptx
```

```typescript
// PDF generation using @react-pdf/renderer
// PPTX generation using pptxgenjs
// Both render the same report data in appropriate formats
```

**Testing:**
- Board report includes all sections: summary, heat map, critical positions, ISO metrics
- Narrative summary is auto-generated text (e.g., "Succession coverage improved from 72% to 78% this quarter...")
- PDF export renders heat map as a colored table image
- PowerPoint export creates one slide per section with charts
- Report is accessible to executive and CHRO roles only
- Trend charts show 4-quarter ISO metrics history
- Report generation completes within 15 seconds

### Task 10.3: Scheduled Reports & Notifications

**What:** Schedule monthly/quarterly report generation and email delivery to configured recipients.

**Design:**

```typescript
// Cron job configuration
// - Monthly: compute ISO 30433 metrics, evaluate risk alerts
// - Quarterly: generate board report, email to CHRO distribution list
// - On HRIS sync: re-evaluate bench strength alerts

// packages/api/src/jobs/scheduled-reports.ts
export const scheduledJobs = [
  { cron: '0 0 1 * *', handler: computeMonthlyMetrics },   // 1st of month
  { cron: '0 0 1 */3 *', handler: generateQuarterlyReport }, // quarterly
];
```

**Testing:**
- Monthly metrics job creates a new `succession_metrics` record for the period
- Quarterly report job generates PDF and sends email to configured recipients
- Email includes report as attachment and summary text in body
- Job failure logs error and sends alert to admin
- Job is idempotent (re-running for same period updates existing metrics, not duplicates)

### Definition of Done — Phase 10
- [ ] ISO 30433 metrics computed correctly for any time period
- [ ] Metrics stored historically for trend analysis
- [ ] Board report auto-generated with all sections
- [ ] PDF and PowerPoint export functional
- [ ] Scheduled report generation and email delivery
- [ ] CSV/JSON export for ISO 30414 compliance

---

## Phase 11: Scenario Modeling & NL Queries

**Goal:** Bench strength scenario modeling (simulate departures, compute coverage impact) and natural language succession queries.

**Duration estimate:** 3-4 weeks

### Task 11.1: Scenario Modeling Engine

**What:** Simulate simultaneous departure scenarios and compute resulting bench strength impact. Answer questions like "What happens if both the CFO and VP Finance leave?"

**Design:**

```python
# apps/ai-service/src/scenarios/scenario_engine.py
@dataclass
class DepartureScenario:
    departing_employees: list[str]  # employee IDs

@dataclass
class ScenarioResult:
    positions_losing_all_successors: list[PositionImpact]
    positions_gaining_single_point_risk: list[PositionImpact]
    cascading_impacts: list[CascadingImpact]  # positions affected transitively
    coverage_rate_before: float
    coverage_rate_after: float
    recommended_actions: list[str]  # AI-generated recommendations

class ScenarioEngine:
    async def simulate(self, scenario: DepartureScenario, tenant_id: str) -> ScenarioResult:
        # 1. Get current bench strength
        # 2. Remove departing employees from all succession plans
        # 3. Recalculate bench strength metrics
        # 4. Identify cascading impacts (departing employee is a successor AND
        #    their departure leaves a position with no remaining successors,
        #    AND that position's current holder is also a successor for another position)
        # 5. Generate recommended actions (cross-train, hire, accelerate development)
        pass
```

```typescript
// API endpoint
// POST /api/v1/scenarios/simulate
// Body: { departingEmployeeIds: ["uuid1", "uuid2"] }
// Response: ScenarioResult
```

**Testing:**
- Simulating departure of an employee who is the only successor for 2 positions correctly identifies both as losing all coverage
- Simulating departure of 3 employees simultaneously shows cascading impacts
- Coverage rate before/after are correctly calculated
- Recommended actions are specific ("Cross-train Employee X for Position Y")
- Simulation does not modify actual data (read-only simulation)
- Scenario with 0 departing employees returns current bench strength unchanged
- Performance: simulation for 200 key positions and 5 departing employees completes in < 3s

### Task 11.2: Natural Language Query Interface

**What:** Allow HR leaders to ask succession questions in natural language (e.g., "Who is ready now for the VP Finance role?", "What are the biggest succession risks in Engineering?").

**Design:**

```python
# apps/ai-service/src/nlp/query_processor.py
from anthropic import Anthropic

class SuccessionQueryProcessor:
    def __init__(self):
        self.client = Anthropic()

    async def process_query(self, query: str, tenant_id: str) -> QueryResult:
        # 1. Use Claude to classify query intent and extract entities
        intent = await self._classify_intent(query)
        # Intents: successor_lookup, risk_assessment, bench_strength,
        #          skills_gap, scenario_simulation, metrics_query

        # 2. Map intent to API calls
        if intent.type == 'successor_lookup':
            position = await self._resolve_position(intent.position_name, tenant_id)
            successors = await self._get_successors(position.id, intent.readiness_filter)
            return QueryResult(
                answer=self._format_successor_list(successors, position),
                data=successors,
                visualisation='successor_table',
            )
        # ... handle other intents

    async def _classify_intent(self, query: str) -> QueryIntent:
        response = await self.client.messages.create(
            model="claude-sonnet-4-20250514",
            system="You are a succession planning query classifier. Extract the intent and entities from the user's question.",
            messages=[{"role": "user", "content": query}],
            # ... structured output schema
        )
        return parse_intent(response)
```

**Testing:**
- "Who is ready now for the CFO role?" returns ready-now successors for CFO position
- "What positions have no successors in Engineering?" returns uncovered positions in Engineering org unit
- "If Sarah Chen leaves, what happens?" triggers scenario simulation for Sarah Chen
- "What is our succession coverage rate?" returns ISO 30433 coverage rate
- Ambiguous query ("Tell me about succession") returns a helpful clarification prompt
- Query with non-existent position name returns "No matching position found"
- Response includes both natural language answer and structured data for UI rendering
- Query processing completes within 5 seconds

### Task 11.3: Scenario & NL Query UI

**What:** Interactive scenario modeling page where users select employees to simulate departure, and view impact analysis. NL query interface as a chat-style input in the navigation bar.

**Design:**

The scenario page features an employee multi-select (search by name), a "Simulate" button, and results panels showing before/after bench strength comparison, impacted positions with risk levels, and recommended actions. The NL query input appears as a persistent search/chat bar accessible from all pages.

**Testing:**
- Selecting 2 employees and clicking "Simulate" shows impact analysis
- Before/after bench strength comparison is visually clear (red/green delta indicators)
- NL query bar auto-completes position names
- Query results display as formatted answers with links to relevant pages
- Error handling for malformed queries shows user-friendly message

### Definition of Done — Phase 11
- [ ] Scenario modeling simulates multi-departure impacts without modifying data
- [ ] Cascading risk analysis identifies transitive succession failures
- [ ] NL queries handle the 5 core query types (successor lookup, risk, bench, gaps, metrics)
- [ ] Results include both natural language and structured data
- [ ] Recommended actions are specific and actionable
- [ ] Query and simulation latency within budget (< 5s)

---

## Phase 12: Self-Hosted Deployment & SOC 2 Readiness

**Goal:** Production-ready self-hosted deployment via Helm chart, security hardening, GDPR DSAR workflow, and SOC 2 Type II preparation.

**Duration estimate:** 3-4 weeks

### Task 12.1: Helm Chart for Kubernetes

**What:** Helm chart that deploys the full stack (PostgreSQL, API server, web frontend, AI service) to any Kubernetes cluster with configurable values for resources, scaling, and storage.

**Design:**

```yaml
# deploy/helm/succession-tool/values.yaml
replicaCount:
  api: 2
  web: 2
  aiService: 1

postgresql:
  enabled: true  # set to false to use external PostgreSQL
  auth:
    database: succession
    username: succession
    password: ""  # set via --set or secrets
  persistence:
    size: 20Gi

api:
  image:
    repository: ghcr.io/succession-planning-tool/api
    tag: latest
  env:
    DATABASE_URL: ""  # templated from postgresql values
    OIDC_ISSUER: ""
    OIDC_CLIENT_ID: ""

web:
  image:
    repository: ghcr.io/succession-planning-tool/web
    tag: latest

aiService:
  image:
    repository: ghcr.io/succession-planning-tool/ai-service
    tag: latest
  resources:
    requests:
      memory: 2Gi  # for embeddings model

ingress:
  enabled: true
  className: nginx
  hosts:
    - host: succession.example.com
```

**Testing:**
- `helm install` on a clean Kubernetes cluster deploys all services
- Health checks pass for all pods within 120 seconds
- PostgreSQL data persists across pod restarts (PVC)
- External PostgreSQL mode works (disable embedded PostgreSQL, provide external URL)
- OIDC configuration connects to Keycloak instance
- Ingress routes traffic correctly to web and API services
- `helm upgrade` applies changes without downtime (rolling update)

### Task 12.2: Security Hardening

**What:** Implement security best practices for enterprise deployment: encryption at rest and in transit, HRIS credential encryption, rate limiting, input validation, and security headers.

**Design:**

```typescript
// packages/api/src/plugins/security.ts
// 1. Rate limiting: 100 requests/minute per user, 1000 requests/minute per tenant
// 2. CORS: configurable allowed origins
// 3. Security headers: HSTS, X-Content-Type-Options, X-Frame-Options, CSP
// 4. Input sanitization: all string inputs validated by Zod schemas
// 5. HRIS credentials: encrypted at rest using AES-256-GCM with key from env var
// 6. Database connections: TLS required in production
// 7. API authentication: JWT with RS256 signing, 1-hour expiry

import { createCipheriv, createDecipheriv, randomBytes } from 'node:crypto';

export function encryptCredentials(plaintext: string, encryptionKey: Buffer): Buffer {
  const iv = randomBytes(16);
  const cipher = createCipheriv('aes-256-gcm', encryptionKey, iv);
  const encrypted = Buffer.concat([cipher.update(plaintext, 'utf8'), cipher.final()]);
  const tag = cipher.getAuthTag();
  return Buffer.concat([iv, tag, encrypted]);
}
```

**Testing:**
- Rate limiting returns 429 when limit is exceeded
- CORS blocks requests from disallowed origins
- All security headers present in HTTP responses
- HRIS credentials stored in database are encrypted (not readable as plaintext)
- Decryption with correct key recovers original credentials
- Database connection fails without TLS in production mode
- SQL injection attempt in API parameter is rejected by Zod validation
- XSS payload in input field is sanitized

### Task 12.3: GDPR Data Subject Access Requests (DSAR)

**What:** Implement workflow for employees to request access to their personal data (succession nominations, talent pool memberships, assessment history, AI recommendations involving them) and request deletion.

**Design:**

```typescript
// packages/api/src/routes/gdpr/dsar.ts
// POST /api/v1/gdpr/access-request — employee requests their data
// GET /api/v1/gdpr/access-request/:id — admin reviews request
// POST /api/v1/gdpr/access-request/:id/export — generate data export
// POST /api/v1/gdpr/access-request/:id/delete — process deletion request

export async function exportEmployeeData(employeeId: string): Promise<EmployeeDataExport> {
  return {
    personalData: await getEmployee(employeeId),
    assessments: await getAssessments(employeeId),
    nominations: await getNominations(employeeId),
    talentPoolMemberships: await getPoolMemberships(employeeId),
    aiRecommendations: await getAIRecommendations(employeeId),
    auditLog: await getAuditEntries(employeeId),
  };
}

export async function deleteEmployeeData(employeeId: string): Promise<DeletionResult> {
  // Soft delete: anonymize personal data, retain statistical records
  // 1. Replace first_name, last_name, email with hashed values
  // 2. Remove demographics JSONB
  // 3. Retain anonymized assessment and nomination records for ISO metrics
  // 4. Log deletion action in audit log
}
```

**Testing:**
- Access request returns all personal data associated with the employee
- Export includes assessments, nominations, pool memberships, and AI recommendations
- Data export is in JSON format, downloadable
- Deletion request anonymizes personal fields (name → hash, email → null)
- Anonymized records still count toward ISO metrics (statistical retention)
- Audit log records the DSAR and deletion action
- 30-day GDPR response deadline is tracked with alerts

### Task 12.4: SOC 2 Evidence Collection

**What:** Implement automated evidence collection for SOC 2 Type II audit: access control reviews, change management logs, incident response procedures, and data backup verification.

**Design:**

```typescript
// packages/api/src/services/compliance/soc2.ts
// Automated evidence:
// 1. Access reviews: quarterly export of user_role assignments
// 2. Change management: git commit log export with PR review evidence
// 3. Audit trail: complete audit_log export for the audit period
// 4. Backup verification: automated backup test restores
// 5. Encryption verification: automated check that all HRIS credentials are encrypted
// 6. Vulnerability scanning: integration with Dependabot/Snyk alerts

// GET /api/v1/admin/compliance/soc2-evidence?period=2026-Q2
```

**Testing:**
- Evidence export generates all required SOC 2 evidence categories
- Access review export includes every user's roles and org unit scope
- Audit trail export is complete (no gaps in coverage)
- Backup verification cron runs monthly and logs success/failure
- Evidence package is downloadable as a ZIP file

### Definition of Done — Phase 12
- [ ] Helm chart deploys successfully on Kubernetes with external PostgreSQL
- [ ] OIDC authentication with Keycloak works in self-hosted deployment
- [ ] All security hardening measures implemented and verified
- [ ] GDPR DSAR workflow: access export and anonymized deletion
- [ ] SOC 2 evidence collection automated for key controls
- [ ] Production deployment checklist documented
- [ ] Load testing: system handles 200 concurrent users with < 500ms p95 latency

---

## Definition of Done (Global)

Every phase must meet these criteria before it is considered complete:

### Code Quality
- [ ] All TypeScript/Python code passes lint with zero warnings
- [ ] TypeScript strict mode enabled with no `@ts-ignore` or `any` types in production code
- [ ] All public API functions have JSDoc/docstring documentation
- [ ] No TODOs left in merged code (tracked as issues instead)

### Testing
- [ ] Unit test coverage >= 80% for business logic (services, matchers, metrics)
- [ ] Integration tests for every API endpoint (happy path + error cases)
- [ ] E2E tests (Playwright) for every critical user flow (login, create plan, nominate, calibrate, generate report)
- [ ] Database migration tested against PostgreSQL 16 in CI

### Security & Compliance
- [ ] No hardcoded secrets in codebase (verified by CI secret scanning)
- [ ] All user input validated via Zod schemas before processing
- [ ] Tenant isolation verified for every new API endpoint
- [ ] RBAC enforced on all routes with appropriate permissions
- [ ] Audit log covers all state-changing operations

### Documentation
- [ ] OpenAPI spec updated and published for all API endpoints
- [ ] Architecture decision records (ADRs) written for significant technical choices
- [ ] CHANGELOG entry for every phase with user-facing description

### Operations
- [ ] Docker images build and run on linux/amd64 and linux/arm64
- [ ] Health check endpoints return correct status for all services
- [ ] Structured logging (JSON) with request correlation IDs
- [ ] Database migrations are reversible (down migration exists)
