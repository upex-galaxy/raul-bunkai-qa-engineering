# Non-Functional Specifications — Bunkai TMS

> Generated: 2026-06-23
> Source: `next.config.ts`, `lib/api/handler.ts`, `lib/api/logging.ts`, `lib/api/idempotency.ts`, `lib/jira/import-runner.ts`, `middleware.ts`, `lib/env.ts`, Supabase migrations.
> Rule: No numbers claimed without evidence. Unverified → Discovery Gap.

---

## NFR Summary

| Category | Implemented? | Maturity | Notes |
|----------|-------------|---------|-------|
| Performance | Partial | Low | Static caching on OpenAPI only; no load testing found |
| Security | Partial | Medium | RLS + PAT hash storage strong; CSP/headers not found |
| Reliability | Partial | Low | Structured logging present; no error boundary, no retry logic |
| Scalability | Partial | Medium | Stateless serverless; Vercel `after()` for async; no queue system |
| Observability | Minimal | Low | Structured JSON logging only; no APM or tracing |

---

## 1. Performance

### NFR-PERF-001: Static OpenAPI Spec Serving

| Aspect | Value |
|--------|-------|
| **Target** | `max-age=300` (5-minute browser cache), `s-maxage=300` (5-minute CDN cache) |
| **Implementation** | `export const dynamic = 'force-static'` + explicit `cache-control` header |
| **Evidence** | `app/api/openapi/route.ts` lines 14, 22 |

The OpenAPI spec at `/api/openapi` is a statically-generated JSON file served with 5-minute browser and CDN caching. This is the only route with explicit caching configuration found in the codebase.

---

### NFR-PERF-002: Jira Import Pagination Safety Ceiling

| Aspect | Value |
|--------|-------|
| **Target** | At most 100,000 issues per import run (100 per page × 1000 pages max) |
| **Implementation** | `PAGE_SIZE = 100`, `MAX_PAGES = 1000` constants |
| **Evidence** | `lib/jira/import-runner.ts` lines 28–29 |

The Jira import worker paginates results in chunks of 100, with a hard ceiling of 1000 pages to prevent non-terminating runs. No Vercel function timeout configured explicitly.

---

### NFR-PERF-003: Full-Text Search on ATCs

| Aspect | Value |
|--------|-------|
| **Target** | [Unknown — no P95 latency target found] |
| **Implementation** | `atcs.tsv` tsvector column with GIN index; maintained by trigger on INSERT/UPDATE |
| **Evidence** | `supabase/migrations/0004_atcs.sql` (tsv column + GIN index) |

PostgreSQL full-text search via `tsvector` allows tag-based and keyword search on ATC titles and content without full-table scans.

---

### NFR-PERF-004: Rate Limiting

| Aspect | Value |
|--------|-------|
| **Target** | [Unknown] |
| **Implementation** | `rate_limited` error code (429) defined but no rate-limit middleware found |
| **Evidence** | `lib/api/error-envelope.ts` line 25 (`rate_limited: 'rate_limited'`); no `@upstash/ratelimit` or similar package in `package.json` |

The error code infrastructure for rate limiting exists but no implementation was found. This is a **Discovery Gap** — the product has no confirmed request throttling.

---

## 2. Security

### NFR-SEC-001: Authentication — Dual Auth Surface

| Aspect | Value |
|--------|-------|
| **Target** | All authenticated API requests validated before handler runs |
| **Implementation** | `withApiHandler` wrapper resolves identity (cookie or Bearer PAT) before every handler; `auth: 'public'` opt-out must be explicit |
| **Evidence** | `lib/api/handler.ts` lines 43–46; `lib/api/principal.ts` (ADR-0001 — cookie/PAT parity guarantee) |

Secure by default: every route is authenticated unless it explicitly declares `auth: 'public'`. Cookie sessions and PAT bearer tokens produce the same `Principal` shape, ensuring no auth-method-specific bypass.

---

### NFR-SEC-002: Authorization — Row-Level Security

| Aspect | Value |
|--------|-------|
| **Target** | No tenant data leaks between workspaces |
| **Implementation** | Supabase RLS on all 24 application tables; tenant boundary = `workspace_members` join; `auth.uid()` from JWT |
| **Evidence** | `supabase/migrations/0005_rls_helpers.sql`; all migration files have RLS policy definitions |

RLS is the single source of authorization truth. Application code (handlers, RPCs) uses RLS-scoped clients — never `createAdminClient()` for user-scoped data access.

---

### NFR-SEC-003: Secret Storage — PAT Hash-Only

| Aspect | Value |
|--------|-------|
| **Target** | Raw PAT secret never readable after creation |
| **Implementation** | SHA-256 of full secret stored in `access_token_secrets` (separate table); raw token returned once in creation response |
| **Evidence** | `lib/api/middleware/bearer.ts` lines 76–78; `supabase/migrations/0011_split_token_secrets.sql` |

The secrets table is split from the tokens table to prevent analytics or read-access roles from harvesting hashes. The bearer middleware hashes the full `prefix + remainder` (not just the remainder — a bug noted and fixed in code comments).

---

### NFR-SEC-004: Non-Disclosure (INV-3)

| Aspect | Value |
|--------|-------|
| **Target** | Cross-workspace resource existence is never revealed |
| **Implementation** | Cross-workspace ATC reference in test chain produces `validation_failed` — same as nonexistent ATC |
| **Evidence** | `supabase/migrations/0024_tests.sql` lines 206–224; `lib/api/error-envelope.ts` |

All auth failures also collapse to uniform 401 (bearer middleware) with no leak of which specific check failed.

---

### NFR-SEC-005: Security Headers

| Aspect | Value |
|--------|-------|
| **Target** | [Unknown — not verifiable from code] |
| **Implementation** | `next.config.ts` has no `headers()` function; no Helmet or CSP found |
| **Evidence** | `next.config.ts` (only `reactStrictMode`, `typedRoutes`, `remotePatterns`, `outputFileTracingRoot`) |

No application-level security headers (CSP, HSTS, X-Frame-Options, etc.) are configured in `next.config.ts`. These may be provided by Vercel platform defaults — **cannot verify from code alone**.

---

### NFR-SEC-006: Environment Variable Validation

| Aspect | Value |
|--------|-------|
| **Target** | Server fails fast at startup if required secrets are missing |
| **Implementation** | `lib/env.ts` Zod schema validates all required env vars at module load; throws with actionable error if invalid |
| **Evidence** | `lib/env.ts` lines 15–59 |

Required vars: `NEXT_PUBLIC_SUPABASE_URL`, `NEXT_PUBLIC_SUPABASE_ANON_KEY`, `SUPABASE_SERVICE_ROLE_KEY`. Optional but validated: `SUPABASE_JWT_SECRET`, `ATLASSIAN_URL/EMAIL/API_TOKEN`.

---

## 3. Reliability

### NFR-REL-001: Structured API Logging

| Aspect | Value |
|--------|-------|
| **Target** | Every API request logged with request_id, method, path, status, duration_ms, user_id, error_code |
| **Implementation** | `logRequest()` emits single-line JSON to stdout; Vercel indexes `console.log` output |
| **Evidence** | `lib/api/logging.ts`; `lib/api/handler.ts` (logging in `withApiHandler`) |

All `/api/v1/` routes wrapped by `withApiHandler` emit structured JSON access logs. Logs include `request_id` (propagated from `x-request-id` header or freshly minted UUID) for error traceability.

---

### NFR-REL-002: Request ID Propagation

| Aspect | Value |
|--------|-------|
| **Target** | Every API response includes `x-request-id` header; every error body includes `request_id` |
| **Implementation** | `getRequestId()` injects on all responses; error envelope includes `request_id` field |
| **Evidence** | `lib/api/request-id.ts`; `lib/api/error-envelope.ts` lines 81–93; `lib/api/handler.ts` |

---

### NFR-REL-003: Error Handling — Centralized Envelope

| Aspect | Value |
|--------|-------|
| **Target** | All API errors return `{ error: { code, message, details?, request_id? } }` |
| **Implementation** | `withApiHandler` catches `ApiError` → `errorResponse()`, `ZodError` → 422 envelope, unknown → 500 with request_id |
| **Evidence** | `lib/api/error-envelope.ts`; `lib/api/handler.ts` |

Error codes are machine-readable (`error.code`) — clients branch on code, not message. 18 error codes defined covering 4xx and 5xx ranges.

---

### NFR-REL-004: Health Check Endpoint

| Aspect | Value |
|--------|-------|
| **Target** | Liveness probe always returns 200 with service metadata |
| **Implementation** | `GET /api/v1/health` returns `{ ok: true, service: 'bunkai-tms', env, ts }` |
| **Evidence** | `app/api/v1/health/route.ts`; `export const dynamic = 'force-dynamic'` |

---

### NFR-REL-005: Error Boundary Coverage

| Aspect | Value |
|--------|-------|
| **Target** | [Unknown] |
| **Implementation** | No `app/error.tsx` found at the root level |
| **Evidence** | Filesystem scan — no `error.tsx` in `app/` root |

Uncaught rendering errors in Server Components will produce an unhandled Next.js error page rather than a graceful recovery. This is a **reliability gap**.

---

### NFR-REL-006: Jira Import Worker Resilience

| Aspect | Value |
|--------|-------|
| **Target** | Import job never left stuck in `running` state |
| **Implementation** | Any throw after job claim sets `status: 'failed'` — never left `running`; per-issue errors append to `errors[]` and import continues |
| **Evidence** | `lib/jira/import-runner.ts` lines 47–56 (atomic claim); Jira auth error → `status: 'failed'` |

Per-issue processing errors are non-fatal (appended to `errors[]`). Only Jira auth failure aborts the entire run.

---

### NFR-REL-007: Idempotency for Unsafe Operations

| Aspect | Value |
|--------|-------|
| **Target** | `POST /api/v1/tests` is idempotent with the same `Idempotency-Key` |
| **Implementation** | `beginIdempotentRequest` / `recordIdempotencyResult` / `discardIdempotencyResult` lifecycle backed by `idempotency_keys` table |
| **Evidence** | `lib/api/idempotency.ts`; `app/api/v1/tests/route.ts` |

Replay returns the stored 201 snapshot without re-executing the business logic. Failed idempotency rows can be retried with the same key after a clean payload.

---

## 4. Scalability

### NFR-SCALE-001: Stateless Application Layer

| Aspect | Value |
|--------|-------|
| **Target** | Horizontal scaling by Vercel serverless — no in-memory state |
| **Implementation** | No in-memory session store, no singleton state; all state in PostgreSQL; active workspace resolved from cookie or request body per request |
| **Evidence** | `lib/api/workspace-cookie.ts` (cookie-based active workspace); `lib/supabase/server.ts` (per-request client) |

---

### NFR-SCALE-002: Async Processing via Vercel after()

| Aspect | Value |
|--------|-------|
| **Target** | Long-running Jira imports do not block HTTP responses |
| **Implementation** | Vercel `after()` runs background work after response is sent; import job status tracked in DB |
| **Evidence** | `app/api/v1/imports/route.ts` (after() call); `lib/jira/import-runner.ts` |

`after()` is a Vercel-specific primitive that runs code after the response is sent without blocking it. Timeout is controlled by the Vercel function timeout — **not explicitly configured**.

---

### NFR-SCALE-003: Database Connection Management

| Aspect | Value |
|--------|-------|
| **Target** | [Unknown] |
| **Implementation** | Supabase managed PostgreSQL with PgBouncer (platform default); no custom pool configuration found |
| **Evidence** | No `DATABASE_POOL_SIZE` or `pgbouncer` config in `lib/` or `next.config.ts` |

Connection pooling is managed by the Supabase platform. No application-level pool configuration was found — this is a **Discovery Gap** for high-concurrency scenarios.

---

## 5. Observability

### NFR-OBS-001: Structured JSON Access Logging

| Aspect | Value |
|--------|-------|
| **Target** | Vercel log drain captures all API requests with structured fields |
| **Implementation** | `console.log/error/warn(JSON.stringify(...))` with `level`, `ts`, `component`, `request_id`, `method`, `path`, `status`, `duration_ms`, `user_id`, `error_code` |
| **Evidence** | `lib/api/logging.ts` |

Vercel captures stdout and indexes structured JSON. Fields are sufficient for debugging individual requests via `request_id`.

---

### NFR-OBS-002: APM / Error Tracking

| Aspect | Value |
|--------|-------|
| **Target** | [Unknown] |
| **Implementation** | No Sentry, Datadog, or OpenTelemetry SDK found |
| **Evidence** | Full `package.json` scan — no `@sentry/*`, `@datadog/*`, `@opentelemetry/*` packages |

No application-level APM or error tracking is configured. This means no automatic error alerting, no transaction traces, and no performance baselines beyond Vercel's built-in function metrics.

---

### NFR-OBS-003: Metrics and Alerting

| Aspect | Value |
|--------|-------|
| **Target** | [Unknown] |
| **Implementation** | None found |
| **Evidence** | No `prom-client`, custom counters, or Vercel Analytics SDK found |

No application-level metrics or alerting system is configured. Health check endpoint (`/api/v1/health`) exists as a liveness probe but has no associated uptime monitoring configuration.

---

### NFR-OBS-004: PAT Usage Tracking

| Aspect | Value |
|--------|-------|
| **Target** | Every PAT authentication updates `last_used_at` for audit purposes |
| **Implementation** | Fire-and-forget `touchLastUsed(tokenId)` — swallows errors to avoid blocking auth |
| **Evidence** | `lib/api/middleware/bearer.ts` lines 104–139 |

Token usage is tracked at the row level in `access_tokens`. The fire-and-forget pattern means a network blip does not fail the request, but `last_used_at` may lag by one request.

---

## 6. Compliance

| Standard | Status | Notes |
|---------|--------|-------|
| GDPR | Needs Review | User email stored in Supabase Auth (`auth.users`). No data retention, export, or deletion policy found in codebase. |
| SOC 2 | Needs Review | Structured logging present; no audit trail for admin actions (beyond `activity_log` table in migrations). |
| HIPAA | Not Applicable | No health data handled. |
| PCI-DSS | Not Applicable | No payment processing. |

---

## 7. Discovery Gaps

| Gap | Where Looked | Impact |
|-----|-------------|--------|
| Rate limiting implementation | `middleware.ts`, `lib/api/handler.ts`, `package.json` | No request throttling confirmed — DoS risk |
| Security headers (CSP, HSTS, X-Frame-Options) | `next.config.ts` | Cannot confirm browser security posture without Vercel config access |
| Vercel function timeout for `after()` | `vercel.json` (not found), `next.config.ts` | Import worker may silently time out on large JQL queries |
| Connection pool configuration | `lib/supabase/`, `dbhub.toml` | Supabase platform default (PgBouncer) — no custom pool size |
| APM / error tracking | `package.json` | No automatic error alerting or distributed tracing |
| Magic-link email provider | `package.json`, `lib/` | `RESEND_API_KEY` in `.env.example` but no Resend call found — delivery channel unconfirmed |
| Root error boundary | `app/` filesystem scan | No `app/error.tsx` — uncaught render errors unhandled |
| GDPR data deletion policy | Full codebase scan | No `/api/v1/me/delete` or similar endpoint found |

---

## 8. QA Relevance

### Testable NFRs

| NFR-ID | Testable? | Suggested Approach |
|--------|----------|-------------------|
| NFR-PERF-001 | Yes | `GET /api/openapi` — verify `cache-control: public, max-age=300` header present |
| NFR-PERF-002 | Yes | Submit JQL that returns 0, 100, 101 issues — verify page boundary behavior |
| NFR-PERF-004 | Not yet | Rate limiting not implemented |
| NFR-SEC-001 | Yes | Hit protected route without auth → confirm 401; with revoked PAT → confirm 401 |
| NFR-SEC-002 | Yes | Use DBHub or direct DB query: member in workspace A cannot read workspace B data |
| NFR-SEC-003 | Yes | Create PAT; attempt to read hash from `access_token_secrets` with non-admin client → RLS block |
| NFR-SEC-004 | Yes | Attempt cross-workspace ATC in test chain → verify `validation_failed` (not `not_found`) |
| NFR-SEC-005 | Partial | Use OWASP ZAP to check response headers on `/api/v1/health` |
| NFR-REL-001 | Yes | Call any `/api/v1/` route; check Vercel logs for structured JSON with all required fields |
| NFR-REL-002 | Yes | Check response headers for `x-request-id`; check error body for `request_id` field |
| NFR-REL-004 | Yes | `GET /api/v1/health` → `{ ok: true, service: 'bunkai-tms' }` — always 200 |
| NFR-REL-007 | Yes | POST `/api/v1/tests` with same `Idempotency-Key` twice → second returns same 201 |
| NFR-SCALE-001 | Inferred | No in-process state — all routes are stateless by construction |
| NFR-OBS-004 | Yes | Issue PAT, make a request, check `access_tokens.last_used_at` updated |

### Suggested Tools

| Tool | Use Case |
|------|---------|
| k6 / Artillery | Load testing on `/api/v1/atcs` and `/api/v1/health` to establish P95 baselines |
| OWASP ZAP | Security header scan; CSP verification; auth bypass probing |
| Supabase Studio / DBHub | RLS parity testing — verify per-role data access |
| Playwright | E2E auth flows, redirect behavior verification |
| Custom API client | Idempotency replay testing; PAT scope enforcement |
