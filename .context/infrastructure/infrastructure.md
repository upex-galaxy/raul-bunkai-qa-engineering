# Infrastructure Mapping

> Target: `upex-bunkai-tms`
> Phase 3 — Infrastructure Mapping
> Generated: 2026-06-24

---

## 1. Overview Diagram

```
+---------------------------+
|       Developer           |
|  bun run dev (port 3000)  |
+------------+--------------+
             |
             | git push → main
             v
+---------------------------+      NO CI/CD configured
|    GitHub Repository      |  ←── HIGH RISK: no automated
|  (no .github/workflows/)  |      test/lint/type-check gate
+------------+--------------+
             |
             | Vercel deploy hook (manual or push-triggered via Vercel dashboard)
             v
+------------------------------------+
|          Vercel Platform           |
|  Build: next build                 |
|  Runtime: Next.js Edge + Node.js   |
+----------+-------------------------+
           |               |
    +------+------+   +----+----------+
    |  Production |   |   Staging     |
    |  (main)     |   |  (preview)    |
    +------+------+   +----+----------+
           |               |
           +-------+-------+
                   |
                   v
    +------------------------------+
    |  Supabase (hosted Postgres)  |
    |  Project: fmbpikzpkafptqximhxn|
    |  Region: unknown (gap)       |
    |  Auth + DB + RLS             |
    +------------------------------+
```

---

## 2. CI/CD Configuration

**Status: NOT CONFIGURED — HIGH RISK**

No `.github/workflows/` directory exists. No other CI platform config files found:
- `.gitlab-ci.yml` — not present
- `azure-pipelines.yml` — not present
- `.circleci/config.yml` — not present
- `Jenkinsfile` — not present

### Risk Assessment

| Risk | Impact |
|------|--------|
| No automated type-check on PR | Type regressions reach production silently |
| No automated lint gate | Code quality enforcement is manual only |
| No automated test execution | RLS isolation tests never run in CI |
| No deploy preview isolation | PRs cannot be validated against a fresh environment |
| No branch protection rules enforced by CI | Main branch unprotected from breaking changes |

### Recommended CI Pipeline (for QA reference)

If/when CI is configured, the expected job sequence would be:

```bash
bun install
bun run types:check      # TypeScript gate
bun run lint:check       # ESLint gate
bun run format:check     # Prettier gate
bun test                 # RLS + integration tests (requires NEXT_PUBLIC_SUPABASE_URL)
```

> **QA Note**: Until CI is in place, all quality gates are manual. The `bun run repo:check` script chains these locally and is the nearest equivalent.

---

## 3. Deployment Configuration

### Hosting Platform: Vercel

- Deployment trigger: Vercel dashboard (manual) or Vercel git integration on `main` push
- Build command (Vercel): `next build` (standard Next.js)
- Output: Next.js App Router server (not static export, not standalone Docker)
- No `vercel.json` found — using Vercel defaults

### Signal Evidence

- `lib/urls.ts` reads `process.env.VERCEL_ENV` to determine environment (`production` / `preview` / local)
- `.agents/project.yaml` (referenced in `project-config.md`) lists staging and production Vercel URLs
- No `Dockerfile`, `fly.toml`, `render.yaml`, `netlify.toml`, `.do/app.yaml` found
- No `vercel.json` in repo root

### Platform-Injected Environment Variables (Vercel)

| Variable | Set by | Value |
|----------|--------|-------|
| `VERCEL_ENV` | Vercel | `production` / `preview` / `development` |
| `NEXT_PUBLIC_VERCEL_URL` | Vercel | Preview deployment URL (e.g. `<project>-<hash>.vercel.app`) |

---

## 4. Environments Matrix

| Environment | URL | Branch / Trigger | Auto-Deploy | Approval |
|-------------|-----|-----------------|-------------|---------|
| Local | `http://localhost:3000` | — | — | — |
| Staging | `https://staging-upexbunkai.vercel.app` | PR preview / Vercel preview env | Yes (Vercel) | None |
| Production | `https://upexbunkai.vercel.app` | `main` push | Yes (Vercel) | None (no manual gate detected) |

> **Critical**: All three environments connect to the **same Supabase project** (`fmbpikzpkafptqximhxn`). There is no separate staging database. Test data created in staging or local dev persists in the shared DB and is visible to production users within the same workspace.

---

## 5. Environment Variables by Environment

| Variable | Local | Staging | Production | Notes |
|----------|-------|---------|------------|-------|
| `NEXT_PUBLIC_SUPABASE_URL` | Same for all | Same for all | Same for all | Single Supabase project |
| `NEXT_PUBLIC_SUPABASE_ANON_KEY` | Same for all | Same for all | Same for all | Single Supabase project |
| `SUPABASE_SERVICE_ROLE_KEY` | `.env` | Vercel env | Vercel env | Server-only |
| `NEXT_PUBLIC_APP_URL` | `http://localhost:3000` | `https://staging-upexbunkai.vercel.app` | `https://upexbunkai.vercel.app` | Per-env, required |
| `SUPABASE_JWT_SECRET` | `.env` | Vercel env | Vercel env | For PAT Bearer auth |
| `ATLASSIAN_*` | `.env` | Not needed | Not needed | Dev/agent tooling only |
| `TAVILY_API_KEY` | `.env` | Not needed | Not needed | Dev/agent tooling only |
| `DBHUB_*` | `.env` | Not needed | Not needed | QA inspector only |
| `RESEND_API_KEY` | `.env` | Vercel env (optional) | Vercel env | If Resend used for email |
| `VERCEL_ENV` | Not set (→ `local`) | `preview` (Vercel-injected) | `production` (Vercel-injected) | Auto-injected by Vercel |

---

## 6. Secrets Management

| Secret | Storage Mechanism | Access Scope | Notes |
|--------|------------------|--------------|-------|
| `SUPABASE_SERVICE_ROLE_KEY` | Vercel Environment Variables | Server-only (never client) | Admin access — bypasses RLS |
| `SUPABASE_JWT_SECRET` | Vercel Environment Variables | Server-only | PAT Bearer auth |
| `NEXT_PUBLIC_SUPABASE_ANON_KEY` | Vercel + local `.env` | Client + server | Browser-safe anon key |
| `ATLASSIAN_API_TOKEN` | Local `.env` only | Dev machine only | Not deployed to Vercel |
| `DBHUB_PASSWORD` | Local `.env` only | Dev machine only | See Jira Epic BK-29 for credentials |
| `RESEND_API_KEY` | Local `.env` + Vercel (optional) | Server-only | Email delivery |
| `SUPABASE_ACCESS_TOKEN` | Local `.env` only | Dev/AI agent only | Supabase MCP admin operations |
| `TAVILY_API_KEY` | Local `.env` only | Dev/AI agent only | Web search MCP |

> **Rotation cadence**: Not documented. Recommend quarterly rotation for service role keys.

---

## 7. Cloud Services

| Service | Provider | Purpose | Access |
|---------|---------|---------|--------|
| Database + Auth | Supabase | PostgreSQL 16, Row-Level Security, Auth (magic-link + password), PAT storage | `NEXT_PUBLIC_SUPABASE_URL` + keys |
| Hosting + CDN | Vercel | Next.js server deployment, edge routing, SSL, CDN | Vercel dashboard / git integration |
| Email (auth) | Supabase Auth (built-in SMTP or Resend) | Magic-link emails, auth confirmations | Supabase Auth config |
| Email (app) | Resend (optional) | Transactional emails | `RESEND_API_KEY` |
| Issue tracker | Atlassian Jira | `upexgalaxy69.atlassian.net`, project key `BK` | `ATLASSIAN_*` env vars |
| Database inspector | DBHub MCP | QA read-only DB access via `qa_inspector_ro` role | `DBHUB_*` env vars |
| Web search | Tavily | AI agent web search MCP | `TAVILY_API_KEY` |
| Workflow automation | n8n | AI workflow automation MCP | `N8N_API_URL` + `N8N_API_KEY` |

---

## 8. Database Infrastructure

| Aspect | Value |
|--------|-------|
| Provider | Supabase (hosted, managed) |
| Engine | PostgreSQL 16 |
| Project ref | `fmbpikzpkafptqximhxn` |
| Region | Unknown (not in codebase — gap) |
| Backups | Supabase automatic daily backups (platform-managed) |
| Connection (pooled) | Port 6543 (transaction pooler) — `POSTGRES_URL` |
| Connection (direct) | Port 5432 (session pooler) — `POSTGRES_URL_NON_POOLING` / `DBHUB_PORT` |
| SSL | Required (`sslmode = "require"` in `dbhub.toml`) |
| RLS | Enabled on all tables; tenant boundary = `workspaces` |
| QA role | `qa_inspector_ro.<project-ref>` — read-only, session pooler |
| Schema migrations | 34 SQL files in `supabase/migrations/`, applied via Supabase MCP |
| Type generation | `bun run types:gen` → `lib/types/supabase.ts` |

### Database Diagram

```
Supabase Project: fmbpikzpkafptqximhxn
  |
  +-- auth schema (Supabase-managed)
  |     auth.users (Supabase managed)
  |
  +-- public schema (app-managed via migrations)
        workspaces            (tenant root)
        workspace_members     (user ↔ workspace relationship)
        projects              (scoped to workspace)
        modules               (scoped to project)
        user_stories          (scoped to module)
        acceptance_criteria   (scoped to user_story)
        atcs                  (Acceptance Test Cases)
        atc_steps             (ATC steps)
        tests                 (test executions)
        test_tags             (test tagging)
        runs                  (test run management)
        access_tokens         (PAT storage)
        import_jobs           (Jira import queue)
        module_activity_log   (audit trail)
        project_environments  (env config per project)
        workspace_invites     (invite system)
```

---

## 9. Infrastructure Resources Diagram

```
+------------------+    HTTPS    +------------------+
|   Browser        |------------>|   Vercel CDN     |
|   (Next.js RSC)  |             |   Edge + Node.js  |
+------------------+             +--------+---------+
                                          |
                     +--------------------+--------------------+
                     |                                         |
              +------+--------+                    +-----------+--------+
              | Next.js API   |                    | Static Assets       |
              | /api/v1/*     |                    | public/openapi.json |
              +------+--------+                    +--------------------+
                     |
                     | Supabase JS client
                     v
              +------+--------+
              |   Supabase    |
              |   PostgREST   |
              |   Auth API    |
              |   PostgreSQL  |
              +---------------+
                     ^
                     | Read-only
              +------+--------+
              |   DBHub MCP   |
              |   (QA tool)   |
              +---------------+
```

---

## 10. Infrastructure as Code (IaC)

**Status: NOT CONFIGURED**

No IaC tooling found:
- `*.tf` (Terraform) — not present
- `Pulumi.yaml` — not present
- `cdk.json` (AWS CDK) — not present
- `serverless.yml` — not present

Infrastructure is fully managed by:
1. **Vercel** — hosting, configured via Vercel dashboard
2. **Supabase** — database + auth, configured via Supabase dashboard + MCP migrations
3. **Git-based schema migrations** — `supabase/migrations/` applied via Supabase MCP

> This is acceptable for the current scale (MVP SaaS). IaC would be needed for multi-region, self-hosted, or enterprise deployments.

---

## 11. Monitoring and Observability

**Status: NOT CONFIGURED**

No monitoring tools detected:
- Sentry — not in `package.json`, no `sentry.client.config.ts` / `sentry.server.config.ts`
- Datadog — not found
- New Relic — not found
- Vercel Analytics — `@vercel/analytics` not in `package.json`
- Vercel Speed Insights — `@vercel/speed-insights` not in `package.json`
- Uptime monitoring — not configured

**Available by default (Vercel platform):**
- Vercel deployment logs (function invocations, build logs)
- Supabase dashboard: DB query logs, Auth logs, API request counts

**Recommendations for QA:**
1. Add Sentry for error tracking — `@sentry/nextjs` integrates with App Router
2. Add Vercel Analytics for Core Web Vitals baseline
3. Configure Supabase alert webhooks for RLS policy errors

---

## 12. Deployment Checklist

### Pre-Deploy

- [ ] `bun run types:check` passes (zero TypeScript errors)
- [ ] `bun run lint:check` passes
- [ ] `bun run format:check` passes
- [ ] All DB migrations applied to target environment via Supabase MCP
- [ ] `bun run types:gen` run after any migration (Supabase type regeneration)
- [ ] `bun run openapi:gen` run if any API route changed
- [ ] Environment variables set in Vercel for target environment

### Post-Deploy

- [ ] `GET /api/v1/health` returns `{ ok: true }`
- [ ] `GET /api/v1` returns version banner
- [ ] `GET /api/openapi` returns valid JSON spec
- [ ] Login flow works (magic-link email delivery verified)
- [ ] Create workspace / project flow works

### Rollback

- Vercel: Use Vercel dashboard → Deployments → Redeploy prior deployment
- Database: Migrations are append-only; no automatic rollback. Requires new compensation migration.

---

## 13. Discovery Gaps

- [ ] **Supabase project region** — not documented anywhere in codebase; check Supabase dashboard
- [ ] **Vercel team / org** — not in codebase; needed for Vercel CLI or deploy scripts
- [ ] **Branch protection rules** — GitHub branch protection for `main` not verifiable from local clone; check GitHub settings
- [ ] **Vercel auto-deploy branch** — assumed `main`; confirm in Vercel project settings
- [ ] **Supabase email provider** — magic-link emails may go through Supabase built-in SMTP (limited to 3/hr in free tier) or a configured Resend integration; not confirmed
- [ ] **Supabase plan / limits** — free tier has 500MB DB, 2GB bandwidth, 50K MAU auth limit; current usage unknown
- [ ] **Preview environment isolation** — staging URL is a named Vercel deployment (not PR-per-deploy); PR previews get auto-generated Vercel URLs (`*.vercel.app`) but share the same DB
- [ ] **Backup / restore tested** — Supabase provides daily backups; restore procedure never documented in codebase
- [ ] **QA read-only role setup** — `qa_inspector_ro` role referenced in `dbhub.toml` must be created in Postgres; migration or setup script for this role not found in `supabase/migrations/`

---

## 14. QA Relevance

### Test Environment Access

| What | How | Blocker |
|------|-----|---------|
| Staging URL | `https://staging-upexbunkai.vercel.app` | None — public URL |
| Test user login | `POST /api/v1/auth/signin` (headless) | Need test credentials from BK-29 |
| DB read (QA inspector) | DBHub MCP via `qa_inspector_ro` role | Need `DBHUB_*` creds from BK-29 |
| API spec | `GET /api/openapi` or `public/openapi.json` | None |
| API docs | `GET /api/docs` (Scalar viewer) | None |
| Auth flow testing | Magic-link requires email delivery | Need email access or password-auth test account |

### CI Integration Points (When CI is added)

| What | Where to wire |
|------|-------------|
| E2E test job | After `next build` (Vercel deploy hook) or against staging URL |
| API contract tests | `GET /api/openapi` as spec source |
| DB state assertion | DBHub MCP (`qa_inspector_ro` role, session pooler port 5432) |
| Auth in E2E | `POST /api/v1/auth/signin` — returns session + PAT in one call |

### Key Testing Constraints

1. **Single shared DB for all envs** — E2E tests MUST clean up created data or use isolated workspaces. No truncate/reset available.
2. **No CI gate** — test results are not blocking deploys today.
3. **`data-testid` coverage is HIGH** — good selector stability for Playwright tests.
4. **PAT auth for API tests** — use `bk_pat_*` Bearer token for REST API tests; avoids cookie management.
5. **Magic-link auth in E2E** — requires email interception. Recommend `POST /api/v1/auth/signin` (password auth) for E2E login instead.
