# Project Configuration

> Project: Bunkai TMS (upex-bunkai-tms)
> Generated: 2026-06-23

## Repositories

| Repository | URL | Branch | Purpose |
|------------|-----|--------|---------|
| upex-bunkai-tms (monorepo) | https://github.com/upex-galaxy/upex-bunkai-tms.git | main | Full-stack web app (Next.js + Supabase API routes + UI) |

Source: `git remote -v` output on target repo.

## Tech Stack

### Frontend
- Framework: Next.js 15 (App Router, `typedRoutes: true`, `reactStrictMode: true`)
- Language: TypeScript 5.9+
- Styling: Tailwind CSS 3.4 + `class-variance-authority` + `tailwind-merge`
- UI Components: Radix UI (Dialog, Dropdown, Tabs, Tooltip), `shadcn/ui` (via `components.json`), `lucide-react`
- Tables: TanStack Table 8.x
- Editor: Monaco Editor (`@monaco-editor/react`)
- API Reference viewer: `@scalar/api-reference-react`
- Markdown: `react-markdown` + `rehype-sanitize` + `remark-gfm`
- State: React Server Components + Client Components (no external state manager found)
- Notifications/Toasts: `sonner`
- Command palette: `cmdk`
- Syntax highlighting: `shiki`

Source: `package.json` lines 44–76.

### Backend
- Framework: Next.js 15 App Router — API routes under `app/api/v1/`
- Language: TypeScript 5.9+
- Runtime: Bun 1.x (scripts, installs, type generation)
- Auth: Supabase Auth + custom PAT (Personal Access Tokens) via `lib/api/middleware/bearer.ts`
- Validation: Zod 4.x (`@asteasolutions/zod-to-openapi` for OpenAPI generation)
- API shape: REST JSON, versioned under `/api/v1/`, idempotency key support
- Note: MVP backend is co-located with frontend (monorepo). Phase 2 target: NestJS separate service.

Source: `package.json` scripts + `.agents/project.yaml` line 16. `app/api/` directory tree.

### Database
- Type: PostgreSQL 16
- Provider: Supabase (managed, project ref `fmbpikzpkafptqximhxn`)
- Access: DBHub MCP (configured in `dbhub.toml`) + Supabase MCP via `SUPABASE_ACCESS_TOKEN`
- Migrations: 24 migrations in `supabase/migrations/` (SQL, manually authored)
- RLS: Row-Level Security enabled on all tables; tenant boundary = `workspaces` table
- Type generation: `bun run types:gen` (→ `lib/supabase-types.ts` — not yet generated in snapshot)

Source: `dbhub.toml`, `supabase/migrations/`, `.agents/project.yaml` lines 26–28.

### Infrastructure
- Cloud: Vercel (production + staging preview environments)
- CI/CD: None detected — no `.github/workflows/` directory exists
- Auth sessions: Supabase SSR (`@supabase/ssr`)
- Monitoring: None detected

Source: `.agents/project.yaml` lines 57–64. No `.github/` directory found on target repo filesystem scan.

## Environments

| Environment | URL | Purpose | Access |
|-------------|-----|---------|--------|
| Local | http://localhost:3000 | Dev | Direct (`bun run dev`) |
| Staging | https://staging-upexbunkai.vercel.app | Pre-prod / QA | Public URL, Supabase auth |
| Production | https://upexbunkai.vercel.app | Live | Public URL, Supabase auth |

Source: `.agents/project.yaml` lines 53–64.

Note: All three environments share the **same Supabase project** (`fmbpikzpkafptqximhxn`) — single-project tenancy for MVP. This is a QA risk: test data written against staging bleeds into production reads.

## Tools and Access

- Issue tracker: Jira (`upexgalaxy69.atlassian.net`) — resolved via `[ISSUE_TRACKER_TOOL]` → `/acli`
- Project key: `BK`
- Database: resolved via `[DB_TOOL]` → DBHub MCP (config in `dbhub.toml`)
- API docs: `app/api/docs/page.tsx` → Scalar viewer at `/api/docs` (OpenAPI spec at `/api/openapi`)
- Docs: Unknown — no Confluence/Notion reference found in codebase

## API Spec Source

| Aspect | Value |
|--------|-------|
| **File** | `public/openapi.json` (committed; regenerated via `bun run openapi:gen`) |
| **Runtime endpoint** | `GET /api/openapi` — force-static, cache-control 5 min |
| **Viewer** | `GET /api/docs` — Scalar API reference viewer |
| **Generator** | `lib/openapi/registry.ts` + `@asteasolutions/zod-to-openapi`; route files register via `registry.registerPath()` |
| **Spec version** | OpenAPI 3.1.0, info.version 0.1.0 |
| **Security schemes** | `bearerAuth` (PAT `bk_pat_*`), `cookieAuth` (Supabase session cookie) |
| **bun run api:sync** | Script should consume `public/openapi.json` to generate `api/openapi-types.ts` — confirm script presence in `package.json` |

Source: `lib/openapi/registry.ts`; `app/api/openapi/route.ts` lines 14–22.

Source: `.agents/project.yaml` lines 28–32. `app/api/docs/page.tsx` presence.

## Credential Keys (`.env.example`)

| Key | Purpose |
|-----|---------|
| `NEXT_PUBLIC_SUPABASE_URL` | Supabase project URL |
| `SUPABASE_PUBLISHABLE_KEY` | Browser-safe Supabase key |
| `SUPABASE_SECRET_KEY` | Server-only Supabase secret |
| `SUPABASE_JWT_SECRET` | Sign/verify custom JWTs |
| `SUPABASE_ACCESS_TOKEN` | MCP control-plane token |
| `POSTGRES_*` | Direct DB connection for scripts/tools |
| `NEXT_PUBLIC_APP_URL` | Auth redirect base URL |
| `ATLASSIAN_URL` / `ATLASSIAN_EMAIL` / `ATLASSIAN_API_TOKEN` | Jira + acli |
| `TAVILY_API_KEY` | Web search MCP |
| `N8N_API_URL` / `N8N_API_KEY` | Workflow automation MCP |
| `RESEND_API_KEY` | Transactional email |
| `DBHUB_*` | DBHub MCP (host, port, db, user, password) |

Source: `.env.example` full file.

## Access Checklist

- [x] Repository read access — confirmed (local clone at `f:\IA\UPEX-Repos\DOJO3-Agentic_QA\upex-bunkai-tms`)
- [ ] Database access (MCP) — requires `DBHUB_*` + `SUPABASE_ACCESS_TOKEN` in `.env`
- [ ] Issue tracker access — requires `ATLASSIAN_URL` + `ATLASSIAN_EMAIL` + `ATLASSIAN_API_TOKEN` in `.env`
- [ ] Staging environment reachable — URL known (`https://staging-upexbunkai.vercel.app`); auth credentials needed
- [ ] CI/CD visibility — no CI workflows detected; deployment is via Vercel dashboard/CLI

## Discovery Gaps

- [ ] Production/staging auth test credentials — not in `.env.example`, must be provided by project team (check Jira Epic BK-29 mentioned in `dbhub.toml` comment)
- [ ] Confluence/Notion knowledge base — no docs URL found in codebase; may be Atlassian Confluence under `upexgalaxy69.atlassian.net`
- [ ] Monitoring/alerting stack — no Sentry/Datadog/Vercel Analytics reference found
- [ ] Shared Supabase single-project risk — confirmed all envs share `fmbpikzpkafptqximhxn`; isolation strategy unknown
- [ ] `bun run types:gen` output (`lib/supabase-types.ts`) — not committed in repo snapshot; generated type file absent
