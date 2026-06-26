# Backend Infrastructure

> Target: `upex-bunkai-tms`
> Phase 3 — Backend Discovery
> Generated: 2026-06-24

---

## 1. Runtime Environment

| Aspect | Value |
|--------|-------|
| Runtime | Bun 1.x (primary) |
| Language | TypeScript 5.9+ |
| Framework | Next.js 15 (App Router) |
| Package Manager | Bun (`bun.lock` present) |
| Node compatibility | `@types/node ^25` — Next.js uses Node under the hood for API routes |
| Module system | ESM (`"type": "module"`) |
| TypeScript target | ES2022 |
| Module resolution | `bundler` (Next.js SWC) |

---

## 2. Package Scripts

| Name | Command | Purpose |
|------|---------|---------|
| `dev` | `next dev` | Local development server (port 3000) |
| `build` | `next build` | Production build — **do NOT run locally per CLAUDE.md Rule 14** |
| `start` | `next start` | Start production server (after build) |
| `typecheck` | `tsc --noEmit` | TypeScript check (alias for `types:check`) |
| `types:check` | `tsc --noEmit` | TypeScript strict check — canonical CI command |
| `types:gen` | `bun scripts/gen-supabase-types.ts` | Generate `lib/types/supabase.ts` from live Supabase schema |
| `lint:check` | `eslint .` | Lint all files |
| `lint:fix` | `eslint --fix .` | Auto-fix lint issues |
| `format:check` | `prettier --check ...` | Check formatting |
| `format:fix` | `prettier --write ...` | Auto-fix formatting |
| `repo:check` | Chains format + lint + types + vars + skills checks | Full pre-commit gate |
| `openapi:gen` | `bun scripts/openapi-gen.ts` | Regenerate `public/openapi.json` from route registrations |
| `openapi:diff` | `bun scripts/openapi-diff.ts` | Diff current spec vs committed |
| `api:sync` | `bun scripts/sync-openapi.ts` | Generate `api/openapi-types.ts` from `public/openapi.json` |
| `jira:sync-issues` | `bun scripts/sync-jira-issues.ts` | Sync Jira issues to `.context/PBI/` |
| `jira:sync-fields` | `bun scripts/sync-jira-fields.ts` | Sync Jira custom field catalog |
| `setup` | `bun cli/doctor.ts --preflight && bun cli/install.ts` | Full repo bootstrap |
| `vars:check` | `bun scripts/lint-vars.ts` | Validate template variable references in skills |
| `vars:env:check` | `bun scripts/check-vars.ts` | Validate env var presence |
| `clean` | `rm -rf node_modules dist .next` | Remove build artifacts |
| `prepare` | `husky` | Install git hooks (auto-runs post-install) |
| `claude` | Wraps `claude` with dotenv-cli | Launch Claude Code with `.env` loaded |

---

## 3. Core Dependencies

| Category | Package | Version | Purpose |
|----------|---------|---------|---------|
| **Framework** | `next` | `^15` | App Router, SSR, API routes |
| **Runtime** | `react` / `react-dom` | `^19` | UI rendering |
| **Auth** | `@supabase/ssr` | `^0.10.3` | Supabase SSR auth — cookie-based session management |
| **Auth** | `@supabase/supabase-js` | `^2.106.0` | Supabase JS client |
| **Validation** | `zod` | `^4.4.3` | Schema validation + env validation |
| **OpenAPI** | `@asteasolutions/zod-to-openapi` | `^8.5.0` | Generate OpenAPI 3.1 spec from Zod schemas |
| **API Docs** | `@scalar/api-reference-react` | `^0.9.38` | Scalar OpenAPI viewer at `/api/docs` |
| **Markdown** | `react-markdown` + `rehype-sanitize` + `remark-gfm` | various | Render markdown content |
| **Tables** | `@tanstack/react-table` | `^8.21.3` | Data tables |
| **UI** | `@radix-ui/react-*` | various | Headless UI primitives |
| **DnD** | `@dnd-kit/*` | various | Drag-and-drop (module/test ordering) |
| **Editor** | `@monaco-editor/react` | `^4.7.0` | Code editor for ATC bodies |
| **Syntax** | `shiki` | `^4.2.0` | Syntax highlighting |
| **Toasts** | `sonner` | `^2.0.7` | Notification toasts |
| **CLI tooling** | `@clack/prompts` + `@inquirer/prompts` | various | Interactive CLI scripts |
| **TypeScript** | `typescript` | `^5.9.3` | (dev) Type checking |
| **Linting** | `@antfu/eslint-config` | `^4.16.0` | (dev) ESLint flat-config preset |
| **Git hooks** | `husky` | `^9.1.7` | (dev) Pre-commit hooks via `lint-staged` |
| **Formatting** | `prettier` | `^3.7.4` | (dev) Code formatting |
| **Styling** | `tailwindcss` | `^3.4` | (dev) Utility-first CSS |

---

## 4. Environment Variables

> Key names only. Never paste actual values. See `.env.example` for full context.

### Required (app will not start without these)

| Key | Format / Example | Purpose |
|-----|-----------------|---------|
| `NEXT_PUBLIC_SUPABASE_URL` | `https://<project-ref>.supabase.co` | Supabase project URL — used by client + server |
| `NEXT_PUBLIC_SUPABASE_ANON_KEY` | Long JWT string | Supabase anon key — `lib/supabase/client.ts` + `middleware.ts` |
| `SUPABASE_SERVICE_ROLE_KEY` | Long JWT string | Server-only, bypasses RLS — used by admin client (`lib/supabase/admin.ts`) |
| `NEXT_PUBLIC_APP_URL` | `http://localhost:3000` (dev) / `https://upexbunkai.vercel.app` (prod) | Auth redirect + invite link base URL |

> **Note**: `lib/env.ts` validates these four at startup via Zod. Missing → hard crash with clear error listing missing keys.

### Optional (have defaults or app degrades gracefully)

| Key | Default | Purpose |
|-----|---------|---------|
| `SUPABASE_JWT_SECRET` | — | Sign/verify custom JWTs for PAT Bearer auth. Required if any route uses `impersonatingClient`. |
| `ATLASSIAN_URL` | — | Jira REST API base URL. Missing = Jira import jobs fail gracefully (error code `jira_unauthorized`) |
| `ATLASSIAN_EMAIL` | — | Jira account email |
| `ATLASSIAN_API_TOKEN` | — | Jira API token |

### External Service (only needed when feature/tool enabled)

| Key | Service | Purpose |
|-----|---------|---------|
| `SUPABASE_ACCESS_TOKEN` | Supabase MCP | Admin control-plane access (migrations, edge functions) |
| `POSTGRES_URL` | Supabase Postgres pooler (port 6543) | Direct SQL access via pooled connection |
| `POSTGRES_URL_NON_POOLING` | Supabase Postgres (port 5432) | Direct SQL access — session pooler |
| `POSTGRES_PRISMA_URL` | Supabase Postgres (pgbouncer) | Prisma-compatible pooled URL |
| `POSTGRES_HOST` | Supabase Postgres | DB hostname |
| `POSTGRES_USER` | Supabase Postgres | DB user (`postgres` by default) |
| `POSTGRES_PASSWORD` | Supabase Postgres | DB password |
| `POSTGRES_DATABASE` | Supabase Postgres | DB name (`postgres` by default) |
| `TAVILY_API_KEY` | Tavily | Web search MCP |
| `N8N_API_URL` / `N8N_API_KEY` | n8n | Workflow automation MCP |
| `RESEND_API_KEY` | Resend | Transactional email |
| `DBHUB_TYPE` | DBHub MCP | DB type (`postgres`) |
| `DBHUB_HOST` | DBHub MCP | DB host for QA inspector |
| `DBHUB_PORT` | DBHub MCP | Port — `5432` (session pooler, required for QA) |
| `DBHUB_DATABASE` | DBHub MCP | Database name |
| `DBHUB_USER` | DBHub MCP | `qa_inspector_ro.<project-ref>` (read-only QA role) |
| `DBHUB_PASSWORD` | DBHub MCP | QA inspector password (see Jira Epic BK-29) |
| `JIRA_PROJECT_KEY` | Jira sync script | Default project key (overrides `.agents/project.yaml`) |
| `JIRA_SYNC_OUTPUT` | Jira sync script | Output dir for synced issues (default: `.context/PBI`) |

---

## 5. Database Configuration

| Aspect | Value |
|--------|-------|
| Type | PostgreSQL 16 |
| Provider | Supabase (hosted, managed) |
| Project ref | `fmbpikzpkafptqximhxn` |
| ORM / Client | Supabase JS client (`@supabase/supabase-js`) with typed `Database` interface |
| Type generation | `bun run types:gen` → generates `lib/types/supabase.ts` |
| Migration tool | Supabase MCP `apply_migration` (not Supabase CLI — see migration README) |
| RLS | Enabled on all tables; tenant boundary = `workspaces` table |
| Multi-tenancy | Workspace-based (`workspace_id` column, enforced by RLS) |
| Connection pooling | Supabase built-in pooler; QA uses session pooler port 5432 |
| QA role | `qa_inspector_ro.<project-ref>` — read-only; credentials in Jira Epic BK-29 |

---

## 6. Migration Files

```
supabase/migrations/
  0001_tenancy.sql               Multi-tenant workspace + RLS foundation
  0002_projects_modules.sql      Projects and modules schema
  0003_authoring.sql             User story authoring
  0004_atcs.sql                  Acceptance Test Cases table
  0005_rls_helpers.sql           RLS policy helper functions
  0006_bootstrap_workspace.sql   Workspace bootstrap on signup
  0007_save_atc.sql              ATC save/upsert
  0008_access_tokens.sql         PAT (Personal Access Token) table + scopes
  0009_cross_cutting.sql         Cross-cutting indexes/triggers
  0010_workspace_invites.sql     Invite system
  0011_split_token_secrets.sql   Token secret split for security
  0012_drop_legacy_token_hashes.sql  Cleanup legacy hash column
  0013_module_description.sql    Module description field
  0014_module_soft_delete.sql    Soft delete for modules
  0015_module_move.sql           Move ATCs between modules
  0016_user_story_uniqueness.sql Uniqueness constraint on user stories
  0017_acceptance_criteria_ordering.sql  AC ordering support
  0018_ready_to_test_gate_fn.sql Ready-to-test gate function
  0019_import_jobs.sql           Jira import job queue
  0020_import_jobs_one_active.sql One active import job constraint
  0021_atc_create_update.sql     ATC create/update functions
  0022_invite_integrity_user_lookup.sql  Invite integrity + user lookup
  0023_module_activity_log.sql   Module activity log
  0024_tests.sql                 Tests (not ATCs — execution layer)
  0025_test_read.sql             Test read RPC
  0026_tests_reorder.sql         Test reorder
  0027_atc_search.sql            ATC full-text search RPC
  0028_atc_duplicate.sql         ATC duplicate RPC
  0029_atc_usage.sql             ATC usage RPC
  0030_test_tags.sql             Test tags
  0031_runs.sql                  Test run management
  0032_project_environments_crud.sql  Project environments
  0033_remediate_bk135_admin_scope.sql  Admin scope remediation (BK-135)
  0034_auth_email_status_rpc.sql Auth email status RPC
```

---

## 7. Migration Commands

> Migrations are applied via the **Supabase MCP** `apply_migration` tool, NOT the Supabase CLI. The Supabase CLI `version` column format is incompatible with this repo's `NNNN_` naming — see `supabase/migrations/README.md` for the ledger convention.

```
# View migration ledger (requires Supabase MCP with SUPABASE_ACCESS_TOKEN set)
# Use the Supabase MCP tool: list_migrations

# Apply a new migration (via Supabase MCP)
# Tool: apply_migration
# Name: NNNN_<slug> (matching the repo file basename without .sql)
# SQL: content of the migration file

# Regenerate Supabase TypeScript types (requires NEXT_PUBLIC_SUPABASE_URL in .env)
bun run types:gen
```

> **No local migration apply command.** Supabase is hosted — migrations go directly to the remote project via MCP. There is no local Supabase Docker setup in this repo.

---

## 8. Build Configuration

| Setting | Value |
|---------|-------|
| Output mode | Default (Next.js server-side) |
| `outputFileTracingRoot` | `path.resolve(import.meta.dirname)` |
| `typedRoutes` | `true` — compile-time route type safety |
| `reactStrictMode` | `true` |
| Image remote patterns | `[]` (no external image hosts allowed) |
| Turbopack | Not enabled (`next dev` without `--turbo`) |
| TypeScript strict | `true` |
| `noEmit` | `true` (types only, no JS output from tsc) |

---

## 9. Local Development Setup

```bash
# 1. Install dependencies
bun install

# 2. Set up environment
cp .env.example .env
# Edit .env — fill in at minimum:
#   NEXT_PUBLIC_SUPABASE_URL=https://<project-ref>.supabase.co
#   NEXT_PUBLIC_SUPABASE_ANON_KEY=<anon key from Supabase dashboard>
#   SUPABASE_SERVICE_ROLE_KEY=<service role key — server only>
#   NEXT_PUBLIC_APP_URL=http://localhost:3000
# Optional for full features:
#   SUPABASE_JWT_SECRET=<jwt secret>
#   ATLASSIAN_URL / ATLASSIAN_EMAIL / ATLASSIAN_API_TOKEN
#   DBHUB_* vars (QA database inspector)

# 3. Generate Supabase TypeScript types (requires NEXT_PUBLIC_SUPABASE_URL)
bun run types:gen

# 4. Start development server
bun run dev
# → http://localhost:3000

# 5. Verify (health check)
curl http://localhost:3000/api/v1/health
# Expected: {"ok":true,"service":"bunkai-tms","env":"local","ts":"<timestamp>"}

# 6. Type-check (no build)
bun run types:check

# 7. Lint
bun run lint:check
```

---

## 10. Auth Architecture (Critical for /adapt-framework)

The app uses **two auth methods** that converge to a single `Principal` type in `lib/api/principal.ts`:

```
Browser / UI session
  → Supabase SSR cookie (middleware.ts refreshes on every request)
  → lib/supabase/server.ts createClient() → supabase.auth.getUser()
  → Principal { via: 'cookie', capabilities: ALL_CAPABILITIES }

API / CLI / agent (headless)
  → Bearer PAT: Authorization: Bearer bk_pat_*
  → lib/api/middleware/bearer.ts → lib/api/principal.ts
  → Mints short-lived user JWT → impersonatingClient(userId)
  → RLS sees auth.uid() = the PAT owner; no RLS bypass
  → Principal { via: 'bearer', capabilities: token.scopes }
```

**Auth flow for browser login:**
1. User submits email → `app/api/v1/auth/magic-link` sends magic-link email via Supabase
2. Email click → `app/auth/callback/route.ts` exchanges OTP code for session
3. Session cookies set; `middleware.ts` refreshes on every request to protected routes
4. Protected prefixes: `/projects`, `/onboarding`
5. Public prefixes: `/login`, `/auth`, `/api/auth`

**Admin client** (`lib/supabase/admin.ts`): uses `SUPABASE_SERVICE_ROLE_KEY`. Bypasses RLS. Used for PAT minting only — never for user data reads.

**PAT scopes**: `atc:read`, `atc:write`, `run:execute`, `workspace:admin`

---

## 11. Health Check Endpoints

| Endpoint | Method | Auth | Response |
|----------|--------|------|----------|
| `GET /api/v1/health` | GET | Public | `{ ok: true, service: "bunkai-tms", env: "local|staging|production", ts: "<ISO>" }` |
| `GET /api/v1` | GET | Public | `{ version: "v1", openapi: "/api/openapi", docs: "/api/docs", status: "live" }` |

---

## 12. In-Repo Tests

The repo contains Bun-native integration tests in `lib/` (not E2E — these are DB/RLS tests):

| File | What it tests |
|------|--------------|
| `lib/tests/rls-isolation.test.ts` | RLS tenant isolation — user A cannot read user B's data |
| `lib/tests/read-isolation.test.ts` | Read isolation across workspaces |
| `lib/tests/reorder.test.ts` | ATC reorder RPC |
| `lib/tests/tags.test.ts` | Test tag operations |
| `lib/atcs/search-isolation.test.ts` | ATC search RLS isolation |
| `lib/atcs/duplicate-rpc.test.ts` | ATC duplicate RPC |
| `lib/atcs/usage-rpc.test.ts` | ATC usage RPC |
| `lib/api/rls-parity.test.ts` | Verifies cookie vs PAT auth return same data |
| `lib/runs/start-run.test.ts` | Test run start RPC |
| `lib/environments/environments-rpc.test.ts` | Project environments RPC |

> These tests run directly against the live Supabase DB (`NEXT_PUBLIC_SUPABASE_URL` + `NEXT_PUBLIC_SUPABASE_ANON_KEY` required). No `bun test` script found in `package.json` — they are run via `bun test <file>` or discovered by the Bun test runner.

---

## 13. Discovery Gaps

- [ ] **No `bun test` script** — `package.json` has no test runner script; Bun test runner is used directly but the exact command CI would use is unknown (no CI configured)
- [ ] **`lib/types/supabase.ts` not committed** — `bun run types:gen` must be run after cloning to get full TypeScript types from the live DB schema
- [ ] **Supabase project ref** `fmbpikzpkafptqximhxn` inferred from `supabase/migrations/README.md` — confirm against `.agents/project.yaml` or Supabase dashboard
- [ ] **Connection pooling config** — pool size and max connections not documented; Supabase default limits apply
- [ ] **Email provider for magic links** — Supabase Auth handles email delivery; actual SMTP provider (Resend vs Supabase default) not confirmed from codebase
- [ ] **Seed mechanism** — no seed script found; first workspace/user bootstrapped via `0006_bootstrap_workspace.sql` trigger on `auth.users` insert
