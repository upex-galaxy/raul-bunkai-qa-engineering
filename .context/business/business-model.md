# Business Model — Bunkai TMS

> **Confidence Level: High** for core entities and features (derived from 24 migration SQL files + app route structure). Medium for commercial/revenue model (no billing code found — community/cloud tiers present in schema but no Stripe integration).
> Generated: 2026-06-23

---

## 1. Problem Statement

Bunkai TMS addresses the gap between how QA teams write and manage Acceptance Test Cases (ATCs) and how those ATCs relate to the User Stories and Acceptance Criteria they validate. Teams working in Jira-centric workflows often maintain test cases in disconnected spreadsheets or external tools, losing the direct traceability link from a User Story's Acceptance Criterion down to the specific test case that covers it.

Source: `supabase/migrations/0004_atcs.sql` lines 2–10 (ATC anchoring moat description), `supabase/migrations/0003_authoring.sql` lines 2–10 (User Story/AC relationship), `.agents/project.yaml` line 11 ("kanji 分解 — the art of breaking down a kata into its applications").

The product provides an opinionated TMS that enforces a structural invariant: every ATC must reference at least one Acceptance Criterion. This prevents "orphan" test cases that cover nothing traceable. Teams can import User Stories directly from Jira (via JQL), bring the associated ACs in, and then author ATCs anchored to those ACs.

Source: `supabase/migrations/0019_import_jobs.sql` lines 1–8 (Jira async import), `lib/jira/import-runner.ts`, `app/(app)/projects/[projectSlug]/import-from-jira-dialog.tsx`.

The system also supports a "Test" entity (capital T) — an ordered chain of ATCs — bridging the gap between individual atomic test cases and full end-to-end test sequences. This models real-world execution flows where a QA engineer runs multiple ATCs in a defined order.

Source: `supabase/migrations/0024_tests.sql` lines 1–14.

---

## 2. Business Model Canvas

### Customer Segments

| Segment | Evidence | Found In |
|---------|----------|---------|
| QA Engineers / SDETs authoring and managing test cases | ATC authoring UI, step/assertion model, tag/search, layer classification (UI/API/Unit) | `app/(app)/projects/[projectSlug]/atcs/`, `supabase/migrations/0004_atcs.sql` |
| QA Leads / Test Managers managing workspaces and teams | Workspace RBAC (viewer/member/admin/owner), workspace invites, member management screen | `supabase/migrations/0001_tenancy.sql`, `app/(app)/workspaces/[id]/members/` |
| Development teams importing Jira stories into TMS | Jira import flow (JQL-based, async), AC extraction from Jira ADF format | `lib/jira/`, `app/(app)/projects/[projectSlug]/import-from-jira-dialog.tsx` |
| AI agents / CLI consumers using the REST API via PATs | Personal Access Token system, scoped PATs (`atc:read`, `atc:write`, `run:execute`, `workspace:admin`) | `supabase/migrations/0008_access_tokens.sql` |

### Value Propositions

| Proposition | Evidence | Found In |
|-------------|----------|---------|
| Enforced AC-to-ATC traceability (anchoring moat) | `atc_acceptance_criteria` M:N table; DB constraint; app-layer validation | `supabase/migrations/0004_atcs.sql` lines 388–397; `lib/atcs/builder-guards.ts` |
| Structured ATC format: steps + assertions + tags + layers | `atc_steps`, `atc_assertions` tables; layer enum `(UI, API, Unit)`; tag-based full-text search | `supabase/migrations/0004_atcs.sql` |
| Jira integration (one-way import of Stories + ACs) | Async import jobs with JQL, ADF-to-Markdown converter, AC extraction | `lib/jira/`, `supabase/migrations/0019_import_jobs.sql` |
| Hierarchical test organization: Workspace → Project → Module tree → Story → ATC | Self-referential module tree (max depth 6), slug-based routing | `supabase/migrations/0002_projects_modules.sql` |
| Test chains (ordered ATC sequences for end-to-end flows) | `tests` + `test_steps` tables; `bunkai_create_test` RPC | `supabase/migrations/0024_tests.sql` |
| PAT-based API access for CLI/AI-agent integration | Token scopes, bearer middleware, token family prefix `bk_pat_` | `supabase/migrations/0008_access_tokens.sql`; `lib/api/middleware/bearer.ts` |
| Built-in API documentation (OpenAPI + Scalar viewer) | `app/api/openapi/route.ts`, `app/api/docs/page.tsx` | Route tree |
| Mind-map visualization of test structure | `app/(app)/projects/[projectSlug]/mind-map-view.tsx` | App route |

### Channels

| Channel | Evidence | Found In |
|---------|----------|---------|
| Web application (browser) | Next.js App Router, auth-gated routes | `middleware.ts`, `app/(app)/` |
| REST API (programmatic / AI agent) | `/api/v1/` routes, PAT bearer auth, OpenAPI spec | `app/api/v1/` |
| Email (magic-link auth, workspace invites) | Magic-link route, invite token system | `app/api/v1/auth/magic-link/route.ts`, `app/api/v1/invites/` |

### Customer Relationships

| Type | Evidence | Found In |
|------|----------|---------|
| Self-service onboarding | Onboarding form (workspace + user setup on first login) | `app/(app)/onboarding/` |
| Multi-user collaboration (workspace-scoped RBAC) | `workspace_members` with role/status; invite system | `supabase/migrations/0001_tenancy.sql` + `0010_workspace_invites.sql` |

### Revenue Streams

| Stream | Evidence | Found In |
|--------|----------|---------|
| Plan tiers: `community`, `cloud`, `enterprise` | `workspaces.plan` column with CHECK constraint | `supabase/migrations/0001_tenancy.sql` line 32 |
| Paid plan gates | Unknown — no billing/Stripe code found | None found |

> **Discovery gap**: plan tier enforcement logic and pricing are not implemented in the codebase snapshot. The `community`/`cloud`/`enterprise` distinction exists at the schema level only.

### Key Resources

| Resource | Evidence | Found In |
|----------|----------|---------|
| Supabase PostgreSQL 16 database (persistence + RLS) | 24 migrations, pgcrypto, tsvector full-text search | `supabase/migrations/` |
| Vercel hosting (production + staging) | `project.yaml` environments, Vercel deploy references | `.agents/project.yaml` lines 57–64 |
| Supabase Auth (passwordless via magic link + email/password) | Auth routes, `@supabase/ssr`, middleware session refresh | `app/api/v1/auth/`, `middleware.ts` |
| OpenAPI specification (auto-generated, Scalar-rendered) | `lib/openapi/registry.ts`, `app/api/openapi/route.ts` | Route + lib |

### Key Activities

| Activity | Evidence | Found In |
|----------|----------|---------|
| ATC authoring (create/edit with steps, assertions, AC links) | Full write route with Zod validation + RPC | `app/(app)/projects/[projectSlug]/atcs/new/`, `app/api/v1/atcs/` |
| User Story + Module management | Module CRUD, User Story CRUD, mind-map view | `app/(app)/projects/[projectSlug]/` |
| Jira import (async JQL-based) | Import job system with polling | `lib/jira/import-runner.ts`, `app/api/v1/imports/` |
| Test chain creation (ordered ATC sequences) | `bunkai_create_test` RPC | `app/api/v1/tests/route.ts` |
| PAT management (issue, revoke) | Token routes | `app/api/v1/tokens/` |
| Workspace + member management | Member list, invite flow | `app/(app)/workspaces/[id]/members/` |

### Key Partners

| Partner | Role | Found In |
|---------|------|---------|
| Supabase | Auth + database + real-time | `package.json` `@supabase/ssr`, `@supabase/supabase-js` |
| Vercel | Hosting + edge network + serverless functions (`after()`) | `.agents/project.yaml` environments; `0019_import_jobs.sql` comment |
| Atlassian / Jira | Source of User Stories + ACs for import | `lib/jira/client.ts` |
| Radix UI + shadcn/ui | Accessible UI primitives | `package.json` `@radix-ui/*`; `components.json` |

### Cost Structure

| Cost | Evidence | Found In |
|------|----------|---------|
| Supabase compute (DB + auth) | Single Supabase project across all envs for MVP | `.agents/project.yaml` db_project_ref repeated |
| Vercel serverless compute | Vercel deployment for Next.js | Environment URLs |
| Jira API calls | Import runner uses Jira REST API | `lib/jira/client.ts` |

---

## 3. Discovery Gaps

- Revenue model mechanics (pricing, billing gates, Stripe): **not implemented** in current codebase
- Production traffic / usage scale: **unknown** (no analytics code found)
- Email service provider for magic-link + invite emails: no Resend/SendGrid explicit call found in app routes (`.env.example` has `RESEND_API_KEY` but no app-code usage found in scan)
- Target SLA / uptime expectations: **unknown**
- Phase 2 features (self-hosted NestJS, pgvector semantic search, React Flow mind-map 3D): **not implemented** yet; referenced in `.agents/project.yaml` backend comments

---

## 4. QA Relevance

| Business Aspect | Testing Implication |
|-----------------|---------------------|
| ATC anchoring moat (must link ≥1 AC) | Test that creating an ATC without AC links is rejected (API 422 + UI guard) |
| Workspace RBAC (viewer read-only; member+ write) | Test every mutation endpoint with each role; confirm 403 on viewer/non-member |
| Jira import (async, JQL-based) | Test import lifecycle: queue → running → completed/failed; test JQL validation; test duplicate handling |
| Single Supabase project across envs | Risk: test data bleeds across envs; isolate test users per workspace during E2E |
| PAT scopes (`atc:read`, `atc:write`, `run:execute`, `workspace:admin`) | Test that each scope gates correctly; test revoked tokens return 401 |
| Magic-link auth (passwordless) | Test magic-link expiry; test redirect-after-login flow |
| Module tree (max depth 6) | Test depth enforcement: depth 7 must fail; test move/rename; test cascade deletes |
| Test chain creation (ordered ATCs) | Test empty chain rejection; test cross-workspace ATC reference rejection (INV-3) |
| Workspace invites (7-day expiry, hash-only stored) | Test expired invite rejection; test accepted invite creates membership |
| Plan tiers (community/cloud/enterprise) | Test workspace creation assigns `community` by default; no billing gate code to test yet |

---

## 5. Sources Used

| Claim | Source File | Lines |
|-------|------------|-------|
| Project name "Bunkai", key "BK", domain bunkai.io | `.agents/project.yaml` | 11–13 |
| Monorepo: backend in `app/api/`, frontend in `app/` | `.agents/project.yaml` | 16–23 |
| Supabase project ref | `.agents/project.yaml` | 56 |
| Workspace plan tiers (community/cloud/enterprise) | `supabase/migrations/0001_tenancy.sql` | 32 |
| RBAC roles (viewer/member/admin/owner) | `supabase/migrations/0001_tenancy.sql` | 43 |
| ATC layers (UI/API/Unit), status enum | `supabase/migrations/0004_atcs.sql` | 59–62 |
| ATC anchoring moat description | `supabase/migrations/0004_atcs.sql` | 2–10 |
| PAT scopes (atc:read, atc:write, run:execute, workspace:admin) | `supabase/migrations/0008_access_tokens.sql` | 35–36 |
| Jira async import, Vercel `after()` | `supabase/migrations/0019_import_jobs.sql` | 1–8 |
| Test chain / Test entity | `supabase/migrations/0024_tests.sql` | 1–14 |
| Auth flow (magic link, protected routes) | `middleware.ts` lines 8–9 |
| Staging/prod URLs | `.agents/project.yaml` | 57–64 |
| No GitHub Actions CI | Filesystem scan — no `.github/workflows/` directory found | — |
| Vercel hosting | `.agents/project.yaml` + environment URLs | 57–64 |
