# Frontend Infrastructure

> Target: `upex-bunkai-tms`
> Phase 3 — Frontend Discovery
> Generated: 2026-06-24

---

## 1. Build Configuration

| Aspect | Value |
|--------|-------|
| Framework | Next.js 15 (App Router) |
| Bundler | SWC (Next.js default) + Webpack (no Turbopack flag) |
| Output mode | Server-side (SSR default for App Router) |
| Router | App Router (`app/` directory) — confirmed, no legacy `pages/` directory |
| TypeScript | Strict mode (`"strict": true`), `noEmit: true`, target ES2022 |
| `typedRoutes` | `true` — compile-time route type checking |
| `reactStrictMode` | `true` |
| Image optimization | `next/image` with `remotePatterns: []` (no external hosts) |
| Turbopack | Not enabled (`next dev` without `--turbo`) |
| Module system | ESM (`"type": "module"`) |

---

## 2. Framework Config Snippet (`next.config.ts`)

```typescript
const config: NextConfig = {
  reactStrictMode: true,
  outputFileTracingRoot: path.resolve(import.meta.dirname),
  typedRoutes: true,
  images: {
    remotePatterns: [],   // no external image domains allowed
  },
};
```

> Minimal config — no custom headers, rewrites, redirects, or i18n. No Turbopack. No `output: 'standalone'`.

---

## 3. TypeScript Path Aliases

Defined in `tsconfig.json` — critical for test import resolution:

| Alias | Resolves to |
|-------|------------|
| `@/*` | `./*` (repo root) |
| `@app/*` | `./app/*` |
| `@components/*` | `./components/*` |
| `@lib/*` | `./lib/*` |

> These aliases must be configured in any Playwright/testing framework config via `moduleNameMapper` or Vite `resolve.alias`.

---

## 4. Client Environment Variables (`NEXT_PUBLIC_*`)

> These are the only vars inlined into the browser bundle. Key names only — no values.

| Key | Usage | Notes |
|-----|-------|-------|
| `NEXT_PUBLIC_SUPABASE_URL` | `lib/supabase/client.ts`, `middleware.ts`, test files | Supabase project URL — browser-safe |
| `NEXT_PUBLIC_SUPABASE_ANON_KEY` | `lib/supabase/client.ts`, `middleware.ts`, test files | Supabase anon key — browser-safe |
| `NEXT_PUBLIC_APP_URL` | `lib/env.ts` (server-side Zod validation, runtime reads) | Auth redirect base URL |

> **Security check**: No secret-looking values use the `NEXT_PUBLIC_` prefix. `SUPABASE_SERVICE_ROLE_KEY` and `SUPABASE_JWT_SECRET` do NOT have the public prefix — correct.

> **Note**: `lib/supabase/client.ts` accesses `NEXT_PUBLIC_*` via static `process.env.NEXT_PUBLIC_SUPABASE_URL` (required for Next.js to inline them at build time). Dynamic access would resolve to `undefined` in the browser bundle.

---

## 5. Environment-Specific Values

| Environment | `NEXT_PUBLIC_APP_URL` | `NEXT_PUBLIC_SUPABASE_URL` |
|-------------|----------------------|---------------------------|
| Local | `http://localhost:3000` | same for all envs (single Supabase project) |
| Staging | `https://staging-upexbunkai.vercel.app` | same for all envs |
| Production | `https://upexbunkai.vercel.app` | same for all envs |

> **Risk**: All environments share a single Supabase project (`fmbpikzpkafptqximhxn`). Test data created in staging is visible in production reads if tenancy (RLS workspace isolation) is not enforced. See `infrastructure.md` for risk details.

---

## 6. Static Assets

```
public/
  openapi.json     OpenAPI 3.1 spec (regenerated via `bun run openapi:gen`)
app/
  apple-icon.png   Apple touch icon
  icon.png         App icon (PNG)
  icon.svg         App icon (SVG)
```

> **Image Handling**: `next/image` is available but `remotePatterns` is empty — all images are local or served from the same origin. No CDN `assetPrefix` configured.

---

## 7. Code Splitting Strategy

| Mechanism | Usage |
|-----------|-------|
| `'use client'` directive | Client components opt-in (e.g. `app/api/docs/page.tsx`, form components) |
| React Server Components (RSC) | Default for all App Router pages — server-side rendering |
| `dynamic(...)` / `React.lazy` | Not found in codebase scan |
| Route-level splitting | Automatic via Next.js App Router segment boundaries |

> Code splitting is **route-segment based** (Next.js default). No manual `dynamic()` imports found. `@scalar/api-reference-react` is loaded as a client component (`'use client'`) due to React effects + stylesheet import requirement.

---

## 8. Performance Configuration

| Aspect | Configuration |
|--------|--------------|
| Font optimization | `next/font/google` — Inter, JetBrains Mono, Noto Serif JP. `display: 'swap'`, preloaded by Next.js |
| Image optimization | `next/image` (built-in), WebP/AVIF auto-conversion. No external hosts allowed. |
| Prefetching | Next.js default link prefetching (App Router) |
| Script optimization | `next/script` not found in codebase — no third-party scripts |
| Bundle analyzer | Not configured (`@next/bundle-analyzer` not in dependencies) |
| Core Web Vitals | No `web-vitals` package; no Lighthouse CI configured |

---

## 9. SEO Configuration

| Aspect | Value |
|--------|-------|
| Metadata API | Next.js App Router `Metadata` type (in `app/layout.tsx`) |
| Title | `Bunkai — Test Management System` |
| Description | `Open-core Test Management System. ATCs, modular tests, full traceability...` |
| `metadataBase` | `new URL('http://localhost:3000')` — **needs update for production** |
| OG tags | Not explicitly set (defaults from `metadataBase`) |
| Sitemap | Not found |
| Robots.txt | Not found in `public/` |

> **Gap**: `metadataBase` is hardcoded to `localhost:3000`. Production/staging environments need this set via `NEXT_PUBLIC_APP_URL` or Vercel env vars for correct OG/canonical URLs.

---

## 10. Routing Architecture

```
app/
├── layout.tsx                    Root layout — fonts, Toaster, dark theme
├── page.tsx                      Root page (likely redirects to /projects)
├── globals.css                   CSS custom properties (design tokens) + Tailwind
├── (app)/                        Authenticated app shell (route group)
│   ├── layout.tsx                Authenticated shell layout
│   ├── onboarding/               Onboarding flow
│   ├── projects/                 Projects list + create
│   │   └── [projectSlug]/        Project detail — modules, ATCs, environments
│   └── workspaces/               Workspace management
├── (auth)/                       Auth flow (route group — no shared layout with app)
│   └── login/                    Login form + magic-link + email-first flow
├── api/
│   ├── v1/                       REST API v1 (versioned)
│   │   ├── route.ts              Discovery: version, openapi, docs, status
│   │   ├── health/               Health check — GET /api/v1/health
│   │   ├── auth/                 signin, signup, magic-link, confirm, check-email
│   │   ├── workspaces/           Workspace CRUD
│   │   ├── projects/             Project CRUD
│   │   ├── modules/              Module operations
│   │   ├── atcs/                 ATC CRUD + search + duplicate
│   │   ├── acceptance-criteria/  AC operations
│   │   ├── user-stories/         User story operations
│   │   ├── tests/                Test operations
│   │   ├── runs/                 Test run management
│   │   ├── environments/         Project environments
│   │   ├── invites/              Workspace invite system
│   │   ├── imports/              Jira import jobs
│   │   ├── tokens/               PAT management
│   │   └── me/                   Current user info
│   ├── openapi/                  Serves public/openapi.json (force-static)
│   └── docs/                     Scalar API reference viewer
├── auth/
│   └── callback/                 Magic-link OTP exchange → session
├── design-tokens/                Internal design token catalog (dev reference)
├── invites/                      Invite accept flow
└── qa/                           QA testability page (credentials + selector guide)
```

---

## 11. State Management

| Layer | Solution |
|-------|---------|
| Server state | React Server Components (fetch in components, no cache lib) |
| Client state | React `useState` / `useReducer` — no external state manager (no Redux, Zustand, Jotai found) |
| Data fetching | Native `fetch` in RSC + Next.js data caching; no TanStack Query/SWR |
| Forms | Uncontrolled components + `FormData` — no react-hook-form found |

---

## 12. Auth Integration Points (Critical for /adapt-framework)

| Aspect | Details |
|--------|---------|
| Auth provider | Supabase Auth (magic-link + password) |
| Client library | `@supabase/ssr` — `createBrowserClient` (client), `createServerClient` (server/middleware) |
| Client-side hook | `lib/supabase/client.ts::createClient()` — per-tab singleton, call `supabase.auth.getUser()` or `supabase.auth.getSession()` |
| Server-side | `lib/supabase/server.ts::createClient()` — cookie-store backed, async |
| Session refresh | `middleware.ts` — runs on every non-static request, calls `supabase.auth.getUser()` |
| Protected routes | `/projects`, `/onboarding` — redirect to `/login?next=<path>` if unauthenticated |
| Auth callback | `GET /auth/callback?code=<otp>` — exchanges code for session |
| E2E test auth | `POST /api/v1/auth/signin` — headless sign-in returns session + PAT in one call |

> **For E2E tests**: Use `POST /api/v1/auth/signin` with `{ email, password }` to get a session and PAT programmatically. This is the authoritative headless login path (built for CLI/agent use).

---

## 13. `data-testid` Coverage

Files with `data-testid` attributes (20+ files found):

| Area | Files | Coverage |
|------|-------|---------|
| Projects list/create | `page.tsx`, `create-project-form.tsx` | Yes |
| Project detail | `project-explorer.tsx`, `project-shell.tsx`, `user-story-form.tsx` | Yes |
| ATC interactions | `atc-search-filter.tsx`, `acceptance-criteria-panel.tsx` | Yes |
| Module operations | `create-module-form.tsx`, `rename-module-form.tsx`, `delete-module-dialog.tsx`, `move-module-dialog.tsx` | Yes |
| Environment ops | `create-environment-form.tsx`, `delete-environment-dialog.tsx`, `rename-environment-form.tsx` | Yes |
| Import | `import-from-jira-dialog.tsx` | Yes |
| Auth | `email-first-form.tsx`, `magic-link-disclosure.tsx` | Yes |
| Other | `test-tag-filter.tsx`, `delete-user-story-dialog.tsx`, `not-found.tsx` | Yes |

> **Coverage level: HIGH** — `data-testid` attributes present across all major interactive components. Consistent use of `data-testid` (not `data-cy` or `id`).

---

## 14. Design System

| Aspect | Details |
|--------|---------|
| UI library | shadcn/ui (`components.json` — style: `new-york`, `rsc: true`) |
| Primitive layer | Radix UI (Dialog, Dropdown, Tabs, Tooltip) |
| Icons | `lucide-react` |
| Styling | Tailwind CSS 3.4 + CSS custom properties (design tokens in `globals.css`) |
| Dark mode | Always dark (`className="dark"` on `<html>`, no theme toggle) |
| Design tokens | Defined in `app/globals.css` as CSS variables; catalogued in `app/design-tokens/page.tsx` |
| Fonts | Inter (sans), JetBrains Mono (mono), Noto Serif JP (accent) — loaded via `next/font/google` |
| Color system | Vermillion accent (`#d9543f`), 6-step surface scale, 5-step foreground scale, signal palette |
| Animation | Custom `status-pulse` + `caret-blink` keyframes in `tailwind.config.ts` |

---

## 15. Browser Support

> Not explicitly configured. Defaults:
- Next.js 15 supports modern browsers (ES2022 target)
- No polyfills configured
- No `browserslist` in `package.json`

---

## 16. Discovery Gaps

- [ ] **`robots.txt` / `sitemap.xml`** — not found in `public/`; may be generated dynamically or omitted intentionally (internal tool)
- [ ] **OG/social metadata** — not set beyond title/description; `metadataBase` hardcoded to localhost
- [ ] **`app/qa/` contents** — QA testability page exists but was not read; may contain test credential hints and selector inventory
- [ ] **Bundle size** — no `@next/bundle-analyzer` configured; bundle size unknown
- [ ] **`app/(app)/workspaces/` contents** — not fully explored
- [ ] **`app/invites/` page** — invite accept flow exists; not read
- [ ] **TanStack Query / SWR** — confirmed absent from `package.json`; all data fetching is native RSC fetch or client-side Supabase subscription
