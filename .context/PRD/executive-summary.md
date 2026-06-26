# Executive Summary — Bunkai TMS

> Generated: 2026-06-23
> Discovery confidence: High (derived from codebase, migrations, UI source).
> Prereqs read: `business-model.md`, `domain-glossary.md`, `project-config.md`

---

## 1. Problem Statement

### The Challenge

QA teams working in Jira-centric workflows routinely author test cases in disconnected spreadsheets or standalone tools. The traceability link from a User Story's Acceptance Criterion down to the test case that actually verifies it is either non-existent or maintained manually. As teams grow, this creates invisible coverage gaps: test cases exist but nobody knows which ACs they cover, and ACs exist with no test case anchored to them.

Source: `supabase/migrations/0004_atcs.sql` lines 2–10 (ATC anchoring moat design rationale); `lib/atcs/builder-guards.ts` (application-layer enforcement).

The second problem is fragmentation of execution modalities. A QA engineer running tests manually, a CI pipeline running automated tests, and an AI agent generating ATCs via the REST API all produce disconnected artifacts with no common schema. Reporting and traceability become impossible at scale.

Source: `app/(auth)/login/page.tsx` lines 8–14 (FEATURE_TICKS listing Manual, Agentic, and CI execution); `lib/api/middleware/bearer.ts` (PAT bearer auth for non-browser callers).

A third problem is the operational gap when importing Jira stories into a test tool. Teams either manually re-type story details into the TMS or export/import CSV files, losing structure (ADF formatting) and traceability links in the process.

Source: `supabase/migrations/0019_import_jobs.sql` lines 1–8; `lib/jira/import-runner.ts`.

### Current Alternatives

No alternative tools are named in the codebase. The login page copy references "spreadsheets" implicitly via the IQL methodology pitch. The product positions itself against generic test management tools (not explicitly named).

---

## 2. Solution Overview

### Product Vision

A test management system that enforces structural traceability from Acceptance Criterion to Acceptance Test Case, supporting manual, agentic, and CI execution on a unified schema.

### Core Capabilities

| # | Feature | Problem Addressed | Evidence |
|---|---------|-------------------|---------|
| 1 | ATC Authoring with enforced AC linkage | Orphan test cases covering nothing | `app/(app)/projects/[projectSlug]/atcs/new/`, `lib/atcs/validation.ts` line 41 |
| 2 | Hierarchical test organization (Workspace → Project → Module → Story → ATC) | Flat, unorganized test repositories | `supabase/migrations/0002_projects_modules.sql`; `app/(app)/projects/[projectSlug]/page.tsx` |
| 3 | Jira import (async, JQL-based) | Manual story re-entry; ADF formatting loss | `lib/jira/import-runner.ts`; `app/(app)/projects/[projectSlug]/import-from-jira-dialog.tsx` |
| 4 | PAT-based REST API (AI agent and CI integration) | Disconnected execution modalities | `app/api/v1/`; `lib/api/middleware/bearer.ts`; `supabase/migrations/0008_access_tokens.sql` |
| 5 | Test chains (ordered ATC sequences) | No way to model end-to-end flows from atomic cases | `supabase/migrations/0024_tests.sql`; `app/api/v1/tests/route.ts` |

### Key Differentiators

- **Anchoring moat**: the only capability verified in code — every ATC must reference at least one AC or the API returns 422. Source: `lib/atcs/validation.ts` line 41 (`acceptance_criterion_ids: z.array(z.string().uuid()).min(1)`).
- **Dual auth surface**: cookie-session for browsers + Bearer PAT for CLI/AI-agent callers; same RLS policies apply to both. Source: `lib/api/principal.ts` (ADR-0001 reference).
- **Built-in OpenAPI specification**: served at `/api/openapi` (static JSON), rendered at `/api/docs` via Scalar viewer. Source: `app/api/openapi/route.ts`; `lib/openapi/registry.ts`.
- **Self-hosted first**: Apache-2.0 license, `docker compose up` referenced in login page footer. Source: `app/(auth)/login/page.tsx` line 132.

---

## 3. Success Metrics

### Tracked Metrics

| Metric | Type | Implementation | Source |
|--------|------|---------------|--------|
| None found | — | No analytics SDK or `track()` call detected | Full codebase scan — no posthog/amplitude/mixpanel reference in `package.json` |

### Inferred KPIs (from features, not real tracking)

| KPI | Derived From |
|-----|-------------|
| ATC creation rate (ATCs authored per workspace per week) | ATC write endpoint exists; no tracking |
| Jira import success rate | `import_jobs` status enum (completed/failed); counters on job row |
| PAT usage (CLI/AI adoption) | `access_tokens.last_used_at` column exists — updated on auth |

### Unknown Metrics

- No analytics SDK configured — product telemetry is fully blind at this stage.
- No uptime / error rate metrics (no Sentry/Datadog found).

---

## 4. Target Users

| Persona | System Role | Primary Need | Evidence |
|---------|------------|-------------|---------|
| QA Engineer | `member` | Author and organize ATCs linked to ACs | `app/(app)/projects/[projectSlug]/atcs/new/`; `lib/atcs/validation.ts` |
| QA Lead / Test Manager | `admin` or `owner` | Manage workspace members, review coverage, invite team | `app/(app)/workspaces/[id]/members/`; `supabase/migrations/0001_tenancy.sql` line 43 |
| Development Team Member | `member` | Import Jira stories and verify test coverage | `app/(app)/projects/[projectSlug]/import-from-jira-dialog.tsx` |
| AI Agent / CLI Consumer | PAT bearer | Programmatic ATC creation and test execution via REST API | `app/api/v1/`; `supabase/migrations/0008_access_tokens.sql` |

Full persona detail → `.context/PRD/user-personas.md`

---

## 5. Product Scope

### What's Included

- Workspace multi-tenancy with RBAC (viewer / member / admin / owner)
- Project → Module tree (max depth 6) → User Story → Acceptance Criterion → ATC hierarchy
- ATC authoring: steps, assertions, tags, layer (UI/API/Unit), AC linkage enforcement
- Jira one-way import: async, JQL-based, ADF→Markdown conversion, AC extraction
- Test chain creation (ordered ATC sequences via `bunkai_create_test` RPC)
- PAT management (issue, revoke with soft-delete)
- Built-in OpenAPI spec + Scalar API reference viewer (`/api/docs`)
- Magic-link passwordless auth
- Workspace invite system (7-day expiry, hash-only stored token)
- Mind-map visualization of test structure
- Full-text search on ATCs (PostgreSQL `tsvector`)
- Software Testability Guide at `/qa` (public, no auth)

### What's Not Included

- OAuth (GitHub/Google auth buttons exist in UI but are disabled — "soon" label): `app/(auth)/login/page.tsx` lines 187–216
- Billing / plan enforcement: `workspaces.plan` column exists but no gating logic found
- CI/CD pipeline: no `.github/workflows/` directory detected
- Test execution runner (status updates exist but no execution engine found)
- NestJS separate backend (referenced in `.agents/project.yaml` as Phase 2 target)

### Future Indicators

- Login page: "OAuth and SSO ship next sprint" — `app/(auth)/login/page.tsx` line 173
- GitHub/Google OAuth buttons disabled with "soon" label
- NestJS + pgvector semantic search referenced in `.agents/project.yaml` comments
- Phase 2 features: light mode, pgvector semantic search, React Flow 3D mind-map

---

## 6. Discovery Gaps

| Gap | Impact | Suggested Source |
|-----|--------|-----------------|
| Analytics / product telemetry | Cannot measure adoption or funnel drop-off | Add Posthog or Segment; check Vercel Analytics dashboard |
| Plan tier gating logic | `community`/`cloud`/`enterprise` schema values exist but no enforcement code found | Interview product team; check Stripe integration plans |
| Email service provider | `RESEND_API_KEY` in `.env.example` but no Resend call found in app code | Grep for `resend` in all lib files; check if magic-link email routes through Supabase Auth built-in |
| ATC status update mechanism | `atcs.status` has 6 values but no UI for changing status found | Search for `status` mutation routes in `app/api/v1/atcs/[id]/` |
| Test execution engine | No execution runner found despite `atcs.status` (unrun/running/pass/fail/blocked/skipped) | Search `app/api/v1/` for a `/runs/` or `/execute/` route |

---

## 7. QA Relevance

### Critical Testing Areas

| Area | Reason |
|------|--------|
| ATC anchoring enforcement | Core business invariant — `acceptance_criterion_ids.min(1)` rejection must be 100% reliable |
| RBAC boundary testing | All mutations must respect viewer (read-only) vs member+ distinction |
| Jira import lifecycle | Async flow (queued→running→completed/failed) is hard to test; concurrency and error handling critical |
| PAT scope enforcement | Each of 4 scopes must gate precisely; revoked tokens must return 401 |
| Magic-link expiry | Passwordless auth — expired or reused links must fail gracefully |
| Idempotency keys | POST /api/v1/tests requires `Idempotency-Key` header — missing key returns 400 |

### Risk Areas

| Risk | Severity |
|------|---------|
| Shared Supabase project across all environments | High — test data bleeds across staging and production |
| No CI pipeline | High — no automated regression guard on merges to main |
| No error boundary at root (`app/error.tsx` absent) | Medium — uncaught render errors may produce blank screens |
| Dual auth parity (cookie vs PAT) — must behave identically under RLS | High — ADR-0001 documents the intent; regression testing required |

---

## 8. Document References

| Document | Status | Path |
|----------|--------|------|
| Domain Glossary | Complete | `.context/business/domain-glossary.md` |
| Business Model | Complete | `.context/business/business-model.md` |
| User Personas | Complete | `.context/PRD/user-personas.md` |
| User Journeys | Complete | `.context/PRD/user-journeys.md` |
| Business Feature Map | Pending | `.context/business/business-feature-map.md` (run `/business-feature-map`) |
| Architecture | Complete | `.context/SRS/architecture.md` |
| Functional Specs | Complete | `.context/SRS/functional-specs.md` |
| Non-Functional Specs | Complete | `.context/SRS/non-functional-specs.md` |
| API Contracts | Delegated | Run `bun run api:sync` — OpenAPI spec at `public/openapi.json`; Scalar viewer at `/api/docs` |
