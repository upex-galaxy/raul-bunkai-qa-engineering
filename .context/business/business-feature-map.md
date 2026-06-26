# Business Feature Map — Bunkai TMS

```
+===========================================================+
|                      BUNKAI TMS                           |
|   Business Feature Map — what the system CAN DO           |
|   API · UI · Integrations · CRUD matrix · QA relevance    |
|   Generated: 2026-06-24                                   |
+===========================================================+
```

> **Companion to**: `.context/business/business-data-map.md` (data-centric view)
> **Sources**: `app/api/v1/` route handlers, `app/` pages, `package.json`,
> `.context/SRS/functional-specs.md`, `.context/PRD/`, Supabase migrations,
> `git log --oneline -30` (target repo `upex-bunkai-tms`)

---

## 1. Inventory Summary

| Category | Features | Status |
|----------|----------|--------|
| Auth & Identity | 5 | Stable / Beta |
| Workspace Management | 3 | Stable |
| Project & Environment | 2 | Stable |
| Test Content Authoring | 7 | Stable |
| Test Composition | 3 | Stable |
| Test Execution | 1 | Stable |
| Jira Integration | 1 | Stable |
| API Access Management | 2 | Stable |
| Developer Tools | 2 | Stable |
| **TOTAL** | **26** | |

| Status | Count | Notes |
|--------|-------|-------|
| Stable | 23 | Confirmed, shipped, covered by migrations |
| Beta | 2 | Password auth (BK-166) — shipped, under smoke tests |
| Planned | 1 | Email delivery provider (unresolved gap) |

---

## 2. Feature Catalog

---

### Domain: Auth & Identity

---

#### Feature: Magic-Link Authentication

| Aspect | Value |
|--------|-------|
| **ID** | FEAT-001 |
| **FR** | FR-001 |
| **Status** | Stable |
| **Endpoints** | `POST /api/v1/auth/magic-link`, `GET /auth/callback` |
| **UI** | `/login` → magic-link form → email → callback |
| **Users** | Public (unauthenticated) |
| **Dependencies** | Supabase Auth (OTP), email delivery provider |
| **Evidence** | `app/api/v1/auth/magic-link/route.ts`, `app/auth/callback/route.ts`, `app/(auth)/login/page.tsx`, `middleware.ts` |

**Capabilities:**
- [x] Submit email → Supabase `signInWithOtp`
- [x] Click magic link → `exchangeCodeForSession` → session JWT cookie
- [x] Middleware auto-refreshes expiring session on every request
- [x] Authenticated users hitting `/login` redirect to `/projects`
- [x] Users with no workspace redirect to `/onboarding`

---

#### Feature: Password Authentication (Email-First Flow)

| Aspect | Value |
|--------|-------|
| **ID** | FEAT-002 |
| **FR** | BK-166 (Beta) |
| **Status** | Beta (shipped, in smoke tests) |
| **Endpoints** | `POST /api/v1/auth/check-email`, `POST /api/v1/auth/signup`, `POST /api/v1/auth/confirm`, `POST /api/v1/auth/signin` |
| **UI** | `/login` → email-first routing → password form or signup |
| **Users** | Public (unauthenticated) |
| **Dependencies** | Supabase Auth (password + OTP verification) |
| **Evidence** | `app/api/v1/auth/signin/route.ts`, `app/api/v1/auth/signup/route.ts`, `app/api/v1/auth/confirm/route.ts`, `app/api/v1/auth/check-email/route.ts` |

**Capabilities:**
- [x] `check-email` → returns `{ exists, confirmed }` for email-first UI routing
- [x] `signup` → provisions account + 6-digit OTP (no auto-confirm, no session, no PAT issued yet)
- [x] `confirm` → verifies OTP → establishes session + mints PAT in one round-trip (headless CLI path)
- [x] `signin` → verifies password → establishes session + mints PAT in one round-trip
- [x] New account enforces `min(8)` password; legacy sign-in accepts `min(6)`
- [x] Uniform 401 on all auth failures (no account-existence leak on signin)
- [x] ADR-0007: `check-email` intentionally exposes existence (accepted enumeration tradeoff)

---

#### Feature: PAT Issuance & Bearer Authentication

| Aspect | Value |
|--------|-------|
| **ID** | FEAT-003 |
| **FR** | FR-003 |
| **Status** | Stable |
| **Endpoints** | `GET /api/v1/tokens`, `POST /api/v1/tokens` |
| **UI** | (API-only — no dedicated UI page found) |
| **Users** | Authenticated (cookie session) |
| **Dependencies** | Supabase (access_tokens + access_token_secrets tables) |
| **Evidence** | `app/api/v1/tokens/route.ts`, `lib/api/middleware/bearer.ts`, `lib/api/pat.ts` |

**Capabilities:**
- [x] Mint token `bk_pat_<12-prefix>.<base64url-secret>` with named scopes
- [x] Scopes: `atc:read`, `atc:write`, `run:execute`, `workspace:admin`
- [x] Store only SHA-256 hash; raw secret returned once in 201 response
- [x] Bearer validation: prefix lookup → SHA-256 compare → revocation check → expiry check
- [x] Uniform 401 on all bearer failures (BR-PAT-002: no leak of which check failed)
- [x] PAT carries only declared scopes; cookie session carries all capabilities
- [x] `workspace:admin` cannot be minted via `signin`/`confirm` (ADR-0005)
- [x] `last_used_at` fire-and-forget update on each successful validation

---

#### Feature: PAT Revocation

| Aspect | Value |
|--------|-------|
| **ID** | FEAT-004 |
| **Status** | Stable |
| **Endpoints** | `DELETE /api/v1/tokens/{id}` |
| **UI** | (API-only — no dedicated UI page found) |
| **Users** | Authenticated (cookie session only — PAT cannot revoke tokens) |
| **Dependencies** | `access_tokens` table (RLS owns-row enforcement) |
| **Evidence** | `app/api/v1/tokens/[id]/route.ts` |

**Capabilities:**
- [x] Soft-revoke: sets `revoked_at = now()` (never hard-deletes — audit trail preserved)
- [x] RLS enforces ownership — a foreign token ID returns 404 (not 403)
- [x] PAT callers blocked from this endpoint (session-only operation per ADR-0001)

---

#### Feature: User Profile & Active Workspace

| Aspect | Value |
|--------|-------|
| **ID** | FEAT-005 |
| **Status** | Stable |
| **Endpoints** | `GET /api/v1/me`, `POST /api/v1/me/active-workspace` |
| **UI** | App shell (identity + role display, BK-86) |
| **Users** | Authenticated (cookie or Bearer) |
| **Dependencies** | `workspaces`, `workspace_members` tables |
| **Evidence** | `app/api/v1/me/route.ts`, `app/api/v1/me/active-workspace/route.ts` |

**Capabilities:**
- [x] Introspect authenticated principal: user summary + workspace list + active workspace
- [x] Works with both cookie sessions and Bearer PATs (unified auth gateway ADR-0001)
- [x] Rotate active workspace via `bk_active_ws` httpOnly cookie (does NOT touch Supabase JWT)
- [x] Resolve caller's RBAC role in active workspace

---

### Domain: Workspace Management

---

#### Feature: Workspace Creation & Management

| Aspect | Value |
|--------|-------|
| **ID** | FEAT-006 |
| **Status** | Stable |
| **Endpoints** | `GET /api/v1/workspaces`, `POST /api/v1/workspaces`, `GET /api/v1/workspaces/{id}`, `PATCH /api/v1/workspaces/{id}` |
| **UI** | `/onboarding` (creation), app shell workspace selector |
| **Users** | Authenticated / Owner+ |
| **Dependencies** | `workspaces`, `workspace_members` tables, `bunkai_bootstrap_workspace` RPC |
| **Evidence** | `app/api/v1/workspaces/route.ts`, `app/api/v1/workspaces/[id]/route.ts`, `app/(app)/onboarding/`, `supabase/migrations/0006_bootstrap_workspace.sql` |

**Capabilities:**
- [x] Create workspace (slug + name) — transactional via `bunkai_bootstrap_workspace` RPC (solves chicken-and-egg owner row)
- [x] Slug: lowercase, digits, hyphens, 3–40 chars; globally unique; reserved slugs blocked
- [x] Default plan: `community` on creation
- [x] List workspaces the caller is an active member of (RLS-filtered)
- [x] Get single workspace details
- [x] Rename workspace (name, max 80 chars)
- [x] First-time user redirected to `/onboarding` when no workspace exists

---

#### Feature: Workspace Invite System

| Aspect | Value |
|--------|-------|
| **ID** | FEAT-007 |
| **FR** | FR-006 |
| **Status** | Stable |
| **Endpoints** | `GET /api/v1/workspaces/{id}/invites`, `POST /api/v1/workspaces/{id}/invites`, `DELETE /api/v1/workspaces/{id}/invites/{inviteId}`, `POST /api/v1/invites/accept` |
| **UI** | `/workspaces/[id]/members` (members page), `/invites/accept` (accept page) |
| **Users** | Admin+ (send/revoke), Public (accept via token) |
| **Dependencies** | `workspace_invites`, `workspace_members` tables |
| **Evidence** | `app/api/v1/workspaces/[id]/invites/`, `app/api/v1/workspaces/[id]/invites/[inviteId]/route.ts`, `app/api/v1/invites/accept/`, `supabase/migrations/0010_workspace_invites.sql` |

**Capabilities:**
- [x] Admin+ sends invite: email + role (`viewer`/`member`/`admin`; `owner` not invitable)
- [x] Token: raw returned once; hash-only stored (mirrors PAT security model)
- [x] Invite expires after 7 days (`expires_at`)
- [x] Invitee accepts raw token → creates `workspace_members` (active) + sets `accepted_at`
- [x] Admin can revoke pending invites (sets `revoked_at`)
- [x] List pending/sent invites for a workspace
- [x] Invitation lifecycle state machine: `pending → accepted | revoked | expired`

---

#### Feature: Workspace Members Management

| Aspect | Value |
|--------|-------|
| **ID** | FEAT-008 |
| **Status** | Stable |
| **Endpoints** | (via workspace invite + RLS-filtered workspace queries) |
| **UI** | `/workspaces/[id]/members` |
| **Users** | Admin+ |
| **Dependencies** | `workspace_members` table, RLS `bunkai_is_workspace_member()` helper |
| **Evidence** | `app/(app)/workspaces/[id]/members/page.tsx`, `supabase/migrations/0001_tenancy.sql` |

**Capabilities:**
- [x] View active members list with roles
- [x] Suspend/reinstate members (admin+) — status: `active ↔ suspended`
- [x] RLS enforces: suspended member has zero data access immediately
- [x] RBAC hierarchy: viewer < member < admin < owner

---

### Domain: Project & Environment

---

#### Feature: Project Management

| Aspect | Value |
|--------|-------|
| **ID** | FEAT-009 |
| **Status** | Stable |
| **Endpoints** | `GET /api/v1/workspaces/{id}/projects`, `POST /api/v1/workspaces/{id}/projects` |
| **UI** | `/projects` (list), project creation form |
| **Users** | Viewer (read), Member+ (create) |
| **Dependencies** | `projects` table |
| **Evidence** | `app/api/v1/workspaces/[id]/projects/route.ts`, `app/(app)/projects/page.tsx`, `app/(app)/projects/create-project-form.tsx` |

**Capabilities:**
- [x] Create project inside a workspace (name + optional description)
- [x] Auto-derive slug from name; unique per workspace; reserved slugs blocked
- [x] List workspace projects (RLS-filtered — members only)
- [x] Project description: max 5120 bytes (Markdown)

---

#### Feature: Project Environment Management

| Aspect | Value |
|--------|-------|
| **ID** | FEAT-010 |
| **FR** | BK-148 |
| **Status** | Stable |
| **Endpoints** | `GET /api/v1/projects/{id}/environments`, `POST /api/v1/projects/{id}/environments`, `PATCH /api/v1/environments/{id}`, `DELETE /api/v1/environments/{id}` |
| **UI** | Project settings / environment management (BK-148) |
| **Users** | Viewer (read), Member+ (write) |
| **Dependencies** | `project_environments` table, `bunkai_create_environment`, `bunkai_rename_environment`, `bunkai_delete_environment` RPCs |
| **Evidence** | `app/api/v1/projects/[id]/environments/route.ts`, `app/api/v1/environments/[id]/route.ts`, `supabase/migrations/0032_project_environments_crud.sql` |

**Capabilities:**
- [x] List project environments (ordered by name, stable)
- [x] Add environment (name: 1–50 chars, case-insensitive unique per project)
- [x] Rename environment (same rules; existing runs keep referencing the ID)
- [x] Delete environment — blocked with 409 when ≥1 run references it

---

### Domain: Test Content Authoring

---

#### Feature: Module Tree Management

| Aspect | Value |
|--------|-------|
| **ID** | FEAT-011 |
| **FR** | FR-007 |
| **Status** | Stable |
| **Endpoints** | `GET /api/v1/projects/{id}/modules`, `POST /api/v1/projects/{id}/modules`, `PATCH /api/v1/modules/{id}`, `DELETE /api/v1/modules/{id}`, `GET /api/v1/modules/{id}/user-stories` |
| **UI** | Project workspace — left-panel module tree, `/projects/[projectSlug]` |
| **Users** | Viewer (read), Member+ (write) |
| **Dependencies** | `modules` table, `bunkai_move_module`, `bunkai_update_module`, `bunkai_archive_module_subtree` RPCs |
| **Evidence** | `app/api/v1/projects/[id]/modules/route.ts`, `app/api/v1/modules/[id]/route.ts`, `supabase/migrations/0002_projects_modules.sql`, `0015_module_move.sql`, `0023_module_activity_log.sql` |

**Capabilities:**
- [x] Create module under parent (or at root) — name 2–80 chars, optional description 500 chars
- [x] Auto-derive path segment from name (slugified); unique per parent
- [x] Tree depth ≤ 6 enforced at DB (CHECK) and RPC (BK-96)
- [x] Move module to new parent — `bunkai_move_module` validates entire subtree depth
- [x] Rename module — cascades path rebuild across all descendants (single transaction)
- [x] Soft-delete module — archives module + all descendants + linked stories/ACs/ATCs
- [x] List user stories in a module
- [x] `archived_at IS NULL` filter hides soft-deleted from all active tree queries
- [x] Deep nesting warning at depth 5

---

#### Feature: User Story Management

| Aspect | Value |
|--------|-------|
| **ID** | FEAT-012 |
| **Status** | Stable |
| **Endpoints** | `POST /api/v1/modules/{id}/user-stories`, `GET /api/v1/user-stories/{id}`, `PATCH /api/v1/user-stories/{id}`, `DELETE /api/v1/user-stories/{id}` |
| **UI** | Project workspace — story panel, `/projects/[projectSlug]` |
| **Users** | Viewer (read), Member+ (write) |
| **Dependencies** | `user_stories` table |
| **Evidence** | `app/api/v1/user-stories/[id]/route.ts`, `app/api/v1/modules/[id]/user-stories/route.ts`, `supabase/migrations/0003_authoring.sql`, `0016_user_story_uniqueness.sql` |

**Capabilities:**
- [x] Create user story in a module (title + optional description Markdown)
- [x] Link to Jira issue via `external_id` (immutable once set) + `external_url`
- [x] `external_id` uniqueness enforced per module
- [x] Status: `draft` (default) or `ready_to_test` (gated: ≥1 active AC required)
- [x] Edit title / description / Jira key (Markdown sanitized on save)
- [x] Soft-archive story (sets `archived_at`)
- [x] Description max 50 KB (same limit as Jira import truncation)

---

#### Feature: Acceptance Criteria Management

| Aspect | Value |
|--------|-------|
| **ID** | FEAT-013 |
| **Status** | Stable |
| **Endpoints** | `GET /api/v1/user-stories/{id}/acceptance-criteria`, `POST /api/v1/user-stories/{id}/acceptance-criteria`, `GET /api/v1/acceptance-criteria/{id}`, `PATCH /api/v1/acceptance-criteria/{id}`, `DELETE /api/v1/acceptance-criteria/{id}` |
| **UI** | Project workspace — AC authoring panel |
| **Users** | Viewer (read), Member+ (write) |
| **Dependencies** | `acceptance_criteria` table, `bunkai_move_acceptance_criterion`, `bunkai_archive_acceptance_criterion` RPCs |
| **Evidence** | `app/api/v1/acceptance-criteria/[id]/route.ts`, `app/api/v1/user-stories/[id]/acceptance-criteria/route.ts`, `supabase/migrations/0017_acceptance_criteria_ordering.sql` |

**Capabilities:**
- [x] Create AC on a user story (title + optional detail description, sortable position)
- [x] Read single AC
- [x] Edit title / detail — Markdown sanitized on save
- [x] Reorder ACs — `bunkai_move_acceptance_criterion` RPC (atomic re-numbering)
- [x] Soft-archive AC — `bunkai_archive_acceptance_criterion` closes position gap; drops story to `draft` if last AC removed
- [x] ACs referenced by ATCs cannot be hard-deleted (ON DELETE RESTRICT on `atc_acceptance_criteria`)

---

#### Feature: ATC Creation with Enforced AC Linkage

| Aspect | Value |
|--------|-------|
| **ID** | FEAT-014 |
| **FR** | FR-002 |
| **Status** | Stable |
| **Endpoints** | `POST /api/v1/atcs` |
| **UI** | `/projects/[projectSlug]/atcs/new` |
| **Users** | Member+ (cookie or Bearer `atc:write`) |
| **Dependencies** | `atcs`, `atc_steps`, `atc_assertions`, `atc_acceptance_criteria` tables; `bunkai_save_atc` RPC |
| **Evidence** | `app/api/v1/atcs/route.ts`, `lib/atcs/validation.ts`, `lib/supabase/rpc.ts` |

**Capabilities:**
- [x] Create ATC with title (3–200 chars), layer (`UI | API | Unit`), tags (max 10), steps (min 1), assertions (optional)
- [x] **Anchoring moat**: `acceptance_criterion_ids` ≥1 UUID required — orphan ATCs impossible
- [x] Step positions strictly increasing from 1 (gaps allowed, e.g. [1, 2, 5])
- [x] Content budget: each step/assertion ≤ 2048 bytes (UTF-8 byte count)
- [x] `bunkai_save_atc` validates cross-entity: AC ∈ user_story_id, module_id ∈ project subtree
- [x] Immutable slug computed from title at creation
- [x] Default status: `unrun`
- [x] Activity event `atc.created` emitted
- [x] Returns 201 `{ atc: {...} }`

**Error codes:**
- `validation_failed` (422) — missing ACs, title too short, too many tags
- `steps_position_invalid` (422) — non-increasing positions
- `ac_outside_user_story` (422) — AC not in the story
- `module_outside_project_subtree` (422) — module cross-project
- `slug_collision` (409)

---

#### Feature: ATC Update & Lifecycle Management

| Aspect | Value |
|--------|-------|
| **ID** | FEAT-015 |
| **Status** | Stable |
| **Endpoints** | `PATCH /api/v1/atcs/{id}` |
| **UI** | `/projects/[projectSlug]/atcs/[atcId]` |
| **Users** | Member+ (cookie or Bearer `atc:write`) |
| **Dependencies** | `atcs`, `atc_steps`, `atc_assertions`, `atc_acceptance_criteria` tables; `bunkai_update_atc` RPC |
| **Evidence** | `app/api/v1/atcs/[id]/route.ts`, `lib/atcs/optimistic-lock.ts` |

**Capabilities:**
- [x] Edit ATC: title, layer, tags, steps, assertions, AC links (PUT-style full replace: omitted children cleared)
- [x] Optimistic locking via `X-If-Match: <version>` — 409 on mismatch (BK-96: custom header avoids Vercel edge rewrite)
- [x] Empty body → 200 no-op (no version bump, no event)
- [x] `user_story_id`, `module_id`, `slug` are immutable
- [x] ATC status lifecycle: `unrun → running → pass | fail | blocked | skipped` (via Run execution)

---

#### Feature: ATC Duplication

| Aspect | Value |
|--------|-------|
| **ID** | FEAT-016 |
| **FR** | BK-23 |
| **Status** | Stable |
| **Endpoints** | `POST /api/v1/atcs/{id}/duplicate` |
| **UI** | Project workspace — context menu "Duplicate ATC" |
| **Users** | Member+ |
| **Dependencies** | `bunkai_duplicate_atc` RPC |
| **Evidence** | `app/api/v1/atcs/[id]/duplicate/route.ts`, `supabase/migrations/0028_atc_duplicate.sql` |

**Capabilities:**
- [x] Clone ATC with all steps, assertions, and AC bindings
- [x] New slug: title-based + `-copy-N` suffix (collision-safe)
- [x] Duplicated ATC starts in `unrun` status
- [x] Target module + user story can differ from source

---

#### Feature: ATC Full-Text Search

| Aspect | Value |
|--------|-------|
| **ID** | FEAT-017 |
| **FR** | BK-20 |
| **Status** | Stable |
| **Endpoints** | `GET /api/v1/atcs/search` |
| **UI** | Projects toolbar filter |
| **Users** | Authenticated (cookie or Bearer `atc:read`) |
| **Dependencies** | `atcs.tsv` tsvector GIN index, `bunkai_search_atcs` RPC |
| **Evidence** | `app/api/v1/atcs/search/route.ts`, `supabase/migrations/0027_atc_search.sql` |

**Capabilities:**
- [x] Full-text search across ATCs by free text
- [x] Filter by tags, layer (`UI | API | Unit`), module, user story
- [x] GIN tsvector index for performance
- [x] Workspace RLS applied — cross-workspace leakage impossible
- [x] Returns 200 `{ atcs: [...] }`

---

#### Feature: ATC Usage Report

| Aspect | Value |
|--------|-------|
| **ID** | FEAT-018 |
| **FR** | BK-22 |
| **Status** | Stable |
| **Endpoints** | `GET /api/v1/atcs/{id}/usage` |
| **UI** | ATC detail view — "Used in N tests" |
| **Users** | Authenticated (cookie or Bearer `atc:read`) |
| **Dependencies** | `test_steps`, `tests` tables; `atcUsage` RPC |
| **Evidence** | `app/api/v1/atcs/[id]/usage/route.ts` |

**Capabilities:**
- [x] Report which Tests chain this ATC and at which positions
- [x] ATC with zero chain references → 200 `{ count: 0, used_in: [] }` (not 404)
- [x] Nonexistent / cross-workspace ATC → uniform 404 (INV-3 non-disclosure)

---

### Domain: Test Composition

---

#### Feature: Test Chain Creation

| Aspect | Value |
|--------|-------|
| **ID** | FEAT-019 |
| **FR** | FR-005 |
| **Status** | Stable |
| **Endpoints** | `POST /api/v1/tests` |
| **UI** | `/projects/[projectSlug]/tests/new` |
| **Users** | Member+ (cookie or Bearer `atc:write`) |
| **Dependencies** | `tests`, `test_steps`, `idempotency_keys` tables; `bunkai_create_test` RPC |
| **Evidence** | `app/api/v1/tests/route.ts`, `supabase/migrations/0024_tests.sql`, `lib/api/idempotency.ts` |

**Capabilities:**
- [x] Create ordered chain of ATCs (title + ordered ATC IDs)
- [x] Chain must have ≥1 ATC (`chain_empty` 422 on empty)
- [x] Same ATC may appear multiple times (no UNIQUE on `(test_id, atc_id)`)
- [x] `Idempotency-Key` header required (`[\w-]{8,128}`) — prevents duplicate chains on retry
- [x] Cross-workspace ATC = nonexistent ATC → same `atc_not_in_workspace` error (INV-3)
- [x] Idempotency replay: same key+payload → return snapshot
- [x] Returns 201 + test body

---

#### Feature: Test Chain Reorder

| Aspect | Value |
|--------|-------|
| **ID** | FEAT-020 |
| **FR** | BK-28 |
| **Status** | Stable |
| **Endpoints** | `GET /api/v1/tests/{id}`, `PATCH /api/v1/tests/{id}/reorder` |
| **UI** | `/projects/[projectSlug]/tests/[testId]` — drag-and-drop chain |
| **Users** | Viewer (read), Member+ (reorder, Bearer `atc:write`) |
| **Dependencies** | `tests`, `test_steps` tables; `bunkai_reorder_test_steps` RPC; `@dnd-kit` |
| **Evidence** | `app/api/v1/tests/[id]/route.ts`, `app/api/v1/tests/[id]/reorder/route.ts`, `lib/atcs/optimistic-lock.ts` |

**Capabilities:**
- [x] Get expanded Test (header + ordered chain of ATCs with steps and assertions)
- [x] Reorder chain: complete new `step_ids` permutation (step_id, not atc_id — chain may repeat)
- [x] Optimistic locking: `X-If-Match: <version>` (lenient — absent header skips; 409 on mismatch)
- [x] No-op detection: unchanged order → no version bump, no event
- [x] Validates: non-empty, no duplicate step references, set equality with existing steps
- [x] INV-3: missing/foreign Test → uniform 404

---

#### Feature: Test Tagging

| Aspect | Value |
|--------|-------|
| **ID** | FEAT-021 |
| **FR** | BK-33 |
| **Status** | Stable |
| **Endpoints** | `PUT /api/v1/tests/{id}/tags` |
| **UI** | Test detail view — tag management |
| **Users** | Member+ (cookie or Bearer `atc:write`) |
| **Dependencies** | `tests` table; `bunkai_set_test_tags` RPC |
| **Evidence** | `app/api/v1/tests/[id]/tags/route.ts` |

**Capabilities:**
- [x] Replace entire tag set (PUT semantics — empty array clears all tags)
- [x] Tags normalized: trim, lowercase reserved, deduplication (in RPC)
- [x] Optimistic locking: `X-If-Match: <version>` (lenient)
- [x] No-op detection: unchanged set → no version bump, no event
- [x] Returns composed Test JSON carrying the updated `tags`

---

### Domain: Test Execution

---

#### Feature: Manual Test Run Execution

| Aspect | Value |
|--------|-------|
| **ID** | FEAT-022 |
| **FR** | BK-34 |
| **Status** | Stable |
| **Endpoints** | `POST /api/v1/runs`, `GET /api/v1/runs/{id}` |
| **UI** | `/projects/[projectSlug]/runs/[runId]` — step-by-step execution view |
| **Users** | Member+ (cookie or Bearer `run:execute`) |
| **Dependencies** | `runs`, `run_atcs`, `run_steps` tables; `bunkai_create_run`, `bunkai_get_run_expanded` RPCs |
| **Evidence** | `app/api/v1/runs/route.ts`, `app/api/v1/runs/[id]/route.ts`, `supabase/migrations/0031_runs.sql` |

**Capabilities:**
- [x] Start a Run: Test + environment (validated in same project) + executor mode (`human | agent | ci`)
- [x] Executor mode derivation: cookie → always `human`; Bearer → body or default `human`
- [x] Run snapshots all ATCs + steps at start (`run_atcs`, `run_steps` — frozen, retroactive ATC edits ignored)
- [x] `Idempotency-Key` required for the HTTP request; domain `start_token` governs 24h dedupe window per Test
- [x] Get expanded Run: header + ordered `run_atcs` with `run_steps`
- [x] Run status lifecycle: `running → passed | failed | aborted`
- [x] Optimistic lock (`version` column) for concurrent verdict conflict prevention (BK-39)
- [x] INV-3: missing/foreign Run → uniform 404

> **Discovery Gap**: Run verdict PATCH route (`PATCH /api/v1/runs/{id}/steps/{stepId}` and `POST /api/v1/runs/{id}/verdict`) not confirmed in file scan — BK-35/39 targets noted in business-data-map but routes not found in `app/api/v1/runs/[id]/route.ts` content. May be implemented via RPC calls directly from UI or not yet shipped.

---

### Domain: Jira Integration

---

#### Feature: Jira Import (Async JQL-based)

| Aspect | Value |
|--------|-------|
| **ID** | FEAT-023 |
| **FR** | FR-004 |
| **Status** | Stable |
| **Endpoints** | `POST /api/v1/imports`, `GET /api/v1/imports/{id}` |
| **UI** | Jira import dialog in project workspace |
| **Users** | Member+ |
| **Dependencies** | `import_jobs`, `user_stories`, `acceptance_criteria` tables; Jira REST API v3; Vercel `after()` |
| **Evidence** | `app/api/v1/imports/route.ts`, `app/api/v1/imports/[id]/route.ts`, `lib/jira/import-runner.ts`, `lib/jira/client.ts`, `lib/jira/extract-acceptance-criteria.ts`, `lib/jira/adf-to-markdown.ts` |

**Capabilities:**
- [x] Enqueue async Jira import job (JQL 1–2000 chars) → 202 immediately
- [x] One active import per project (partial unique index) → 409 `import_in_progress` on conflict
- [x] Background worker (Vercel `after()`): pages through Jira (PAGE_SIZE=100, MAX_PAGES=1000 theoretical max)
- [x] Per-issue: ADF → Markdown conversion, AC extraction, module routing (component match or "Inbox")
- [x] UPSERT idempotency keyed on `external_id` (Jira issue key) — re-import same JQL = update not duplicate
- [x] Description truncation at 50 KB with marker
- [x] Import status polling via `GET /api/v1/imports/{id}`
- [x] Job status lifecycle: `queued → running → completed | failed`
- [x] Per-issue error accumulation in `errors[]` JSONB array
- [x] Jira auth error fails entire job (terminal — no retry mechanism)

---

### Domain: API Access Management

---

#### Feature: OpenAPI Specification & Docs

| Aspect | Value |
|--------|-------|
| **ID** | FEAT-024 |
| **Status** | Stable |
| **Endpoints** | `GET /api/openapi`, `GET /api/docs`, `GET /api/v1` |
| **UI** | `/api/docs` — Scalar API reference viewer |
| **Users** | Public |
| **Dependencies** | `@scalar/api-reference-react`, `@asteasolutions/zod-to-openapi`, `public/openapi.json` |
| **Evidence** | `app/api/openapi/route.ts`, `app/api/docs/page.tsx`, `lib/openapi/registry.ts`, `scripts/openapi-gen.ts` |

**Capabilities:**
- [x] `GET /api/openapi` → pre-generated OpenAPI 3.1 JSON (force-static, 5-min cache)
- [x] `GET /api/docs` → Scalar interactive API reference viewer
- [x] `GET /api/v1` → API index (version, openapi path, docs path)
- [x] Spec generated from Zod schemas via `bun run openapi:gen`

---

#### Feature: Health Check

| Aspect | Value |
|--------|-------|
| **ID** | FEAT-025 |
| **Status** | Stable |
| **Endpoints** | `GET /api/v1/health` |
| **UI** | None |
| **Users** | Public |
| **Dependencies** | None |
| **Evidence** | `app/api/v1/health/route.ts` |

**Capabilities:**
- [x] Returns `{ ok, service, env, ts }` — service liveness check (no auth required)

---

## 3. CRUD Matrix

| Entity | Create | Read | Update | Delete | Notes |
|--------|--------|------|--------|--------|-------|
| `Workspace` | ✅ POST | ✅ GET | ✅ PATCH (rename) | ❌ Not available | No delete route found |
| `WorkspaceMember` | ⚠️ Via invite accept | ✅ GET (members page) | ⚠️ Suspend/reinstate | ❌ Not found | Soft status change only |
| `WorkspaceInvite` | ✅ POST | ✅ GET | ❌ No edit | ✅ DELETE (revoke) | Revoke = soft (revoked_at) |
| `Project` | ✅ POST | ✅ GET | ❌ Not found | ❌ Not found | No PATCH /projects/{id} route confirmed |
| `ProjectEnvironment` | ✅ POST | ✅ GET | ✅ PATCH (rename) | ✅ DELETE (hard — blocked if used) | |
| `Module` | ✅ POST | ✅ GET | ✅ PATCH (rename/move) | ⚠️ DELETE (soft archive) | Cascades archive to descendants |
| `UserStory` | ✅ POST | ✅ GET | ✅ PATCH | ⚠️ DELETE (soft archive) | `external_id` immutable once set |
| `AcceptanceCriterion` | ✅ POST | ✅ GET | ✅ PATCH | ⚠️ DELETE (soft archive) | Cannot delete if ATC references it |
| `ATC` | ✅ POST | ⚠️ via search/usage | ✅ PATCH | ❌ Not found | No soft-delete/archive route confirmed |
| `Test` | ✅ POST | ✅ GET | ⚠️ Reorder + Tags only | ❌ Not found | No PATCH title / DELETE route confirmed |
| `TestStep` | (via Test) | (via Test GET) | ✅ PATCH reorder | (via Test) | Part of chain, no standalone CRUD |
| `Run` | ✅ POST | ✅ GET | ⚠️ Step results (gap) | ❌ Not found | Verdict PATCH route not confirmed |
| `AccessToken` | ✅ POST | ✅ GET | ❌ No edit | ✅ DELETE (soft revoke) | Raw token returned once only |
| `ImportJob` | ✅ POST | ✅ GET | ❌ No edit | ❌ No delete | Async job lifecycle |

Legend: ✅ Full, ⚠️ Partial/conditional, ❌ Not available

---

## 4. API Endpoint Inventory

### Auth

| Method | Endpoint | Purpose | Auth |
|--------|----------|---------|------|
| POST | `/api/v1/auth/magic-link` | Send magic-link email | Public |
| POST | `/api/v1/auth/check-email` | Email-first routing helper | Public |
| POST | `/api/v1/auth/signup` | Create account + send OTP | Public |
| POST | `/api/v1/auth/confirm` | Verify OTP + mint PAT + establish session | Public |
| POST | `/api/v1/auth/signin` | Password signin + mint PAT | Public |
| GET | `/auth/callback` | Supabase Auth OAuth callback | Public |

### User Profile

| Method | Endpoint | Purpose | Auth |
|--------|----------|---------|------|
| GET | `/api/v1/me` | Introspect principal + workspaces | Required |
| POST | `/api/v1/me/active-workspace` | Switch active workspace cookie | Required |

### Workspace

| Method | Endpoint | Purpose | Auth |
|--------|----------|---------|------|
| GET | `/api/v1/workspaces` | List caller's workspaces | Required |
| POST | `/api/v1/workspaces` | Create workspace | Required |
| GET | `/api/v1/workspaces/{id}` | Get workspace | Required |
| PATCH | `/api/v1/workspaces/{id}` | Rename workspace | Required (member+) |
| GET | `/api/v1/workspaces/{id}/invites` | List workspace invites | Required (admin+) |
| POST | `/api/v1/workspaces/{id}/invites` | Send invite | Required (admin+) |
| DELETE | `/api/v1/workspaces/{id}/invites/{inviteId}` | Revoke invite | Required (admin+) |
| POST | `/api/v1/invites/accept` | Accept invite token | Public |

### Project

| Method | Endpoint | Purpose | Auth |
|--------|----------|---------|------|
| GET | `/api/v1/workspaces/{id}/projects` | List workspace projects | Required |
| POST | `/api/v1/workspaces/{id}/projects` | Create project | Required (member+) |
| GET | `/api/v1/projects/{id}/environments` | List environments | Required |
| POST | `/api/v1/projects/{id}/environments` | Add environment | Required (member+) |
| PATCH | `/api/v1/environments/{id}` | Rename environment | Required (member+) |
| DELETE | `/api/v1/environments/{id}` | Remove environment | Required (member+) |

### Module

| Method | Endpoint | Purpose | Auth |
|--------|----------|---------|------|
| GET | `/api/v1/projects/{id}/modules` | List project modules | Required |
| POST | `/api/v1/projects/{id}/modules` | Create module | Required (member+) |
| PATCH | `/api/v1/modules/{id}` | Rename/update module | Required (member+) |
| DELETE | `/api/v1/modules/{id}` | Soft-archive module | Required (member+) |
| GET | `/api/v1/modules/{id}/user-stories` | List user stories in module | Required |

### User Story

| Method | Endpoint | Purpose | Auth |
|--------|----------|---------|------|
| POST | `/api/v1/modules/{id}/user-stories` | Create user story | Required (member+) |
| GET | `/api/v1/user-stories/{id}` | Get user story | Required |
| PATCH | `/api/v1/user-stories/{id}` | Update user story | Required (member+) |
| DELETE | `/api/v1/user-stories/{id}` | Soft-archive user story | Required (member+) |
| GET | `/api/v1/user-stories/{id}/acceptance-criteria` | List ACs | Required |
| POST | `/api/v1/user-stories/{id}/acceptance-criteria` | Create AC | Required (member+) |

### Acceptance Criteria

| Method | Endpoint | Purpose | Auth |
|--------|----------|---------|------|
| GET | `/api/v1/acceptance-criteria/{id}` | Get AC | Required |
| PATCH | `/api/v1/acceptance-criteria/{id}` | Update / reorder AC | Required (member+) |
| DELETE | `/api/v1/acceptance-criteria/{id}` | Soft-archive AC | Required (member+) |

### ATCs

| Method | Endpoint | Purpose | Auth |
|--------|----------|---------|------|
| POST | `/api/v1/atcs` | Create ATC | Required (member+ / `atc:write`) |
| GET | `/api/v1/atcs/search` | Full-text ATC search | Required (`atc:read`) |
| PATCH | `/api/v1/atcs/{id}` | Update ATC | Required (member+ / `atc:write`) |
| POST | `/api/v1/atcs/{id}/duplicate` | Duplicate ATC | Required (member+) |
| GET | `/api/v1/atcs/{id}/usage` | ATC usage in tests | Required (`atc:read`) |

### Tests (Chains)

| Method | Endpoint | Purpose | Auth |
|--------|----------|---------|------|
| POST | `/api/v1/tests` | Create test chain | Required (member+ / `atc:write`) |
| GET | `/api/v1/tests/{id}` | Get expanded test | Required |
| PATCH | `/api/v1/tests/{id}/reorder` | Reorder ATC chain | Required (member+ / `atc:write`) |
| PUT | `/api/v1/tests/{id}/tags` | Replace test tags | Required (member+ / `atc:write`) |

### Runs

| Method | Endpoint | Purpose | Auth |
|--------|----------|---------|------|
| POST | `/api/v1/runs` | Start test run | Required (member+ / `run:execute`) |
| GET | `/api/v1/runs/{id}` | Get expanded run | Required |

### Imports (Jira)

| Method | Endpoint | Purpose | Auth |
|--------|----------|---------|------|
| POST | `/api/v1/imports` | Enqueue Jira import | Required (member+) |
| GET | `/api/v1/imports/{id}` | Check import job status | Required |

### Tokens (PAT)

| Method | Endpoint | Purpose | Auth |
|--------|----------|---------|------|
| GET | `/api/v1/tokens` | List caller's PATs | Required |
| POST | `/api/v1/tokens` | Create PAT | Required (cookie) |
| DELETE | `/api/v1/tokens/{id}` | Revoke PAT (soft) | Required (cookie only — no bearer) |

### Developer Tools

| Method | Endpoint | Purpose | Auth |
|--------|----------|---------|------|
| GET | `/api/v1` | API version index | Public |
| GET | `/api/v1/health` | Health check | Public |
| GET | `/api/openapi` | OpenAPI 3.1 spec | Public |

---

## 5. UI Component Inventory

### Pages

| Route | Purpose | Primary Actions |
|-------|---------|-----------------|
| `/` | Root landing / redirect | → `/projects` or `/login` |
| `/login` | Auth entry — magic-link or password | Email submit, magic-link, password |
| `/auth/callback` | Supabase OAuth callback (no UI) | Session exchange |
| `/invites/accept` | Invite acceptance flow | Token input + accept |
| `/onboarding` | Workspace creation (first-time) | Slug + name form |
| `/projects` | Projects list | Create project, select project |
| `/projects/[slug]` | Project workspace | Module tree + story + AC + ATC panels |
| `/projects/[slug]/atcs/new` | ATC creation form | Title, layer, steps, assertions, AC links |
| `/projects/[slug]/atcs/[id]` | ATC detail + edit | Edit ATC fields, duplicate, view usage |
| `/projects/[slug]/tests/new` | Test chain creation | Title, ATC picker |
| `/projects/[slug]/tests/[id]` | Test chain detail + reorder | Drag-and-drop chain, tag management |
| `/projects/[slug]/runs/[id]` | Run execution view | Step-by-step execution |
| `/workspaces/[id]/members` | Members management | Invite, suspend, manage |
| `/api/docs` | Scalar API reference | Interactive try-it-out |

### Key UI Components

| Component | Function |
|-----------|----------|
| Module tree panel | Hierarchical tree with drag-and-drop (dnd-kit) |
| ATC step editor | Monaco-powered rich step/assertion editor |
| Jira import dialog | JQL input + job status polling |
| Tag management | Multi-tag input with normalize/dedupe |
| Members table | TanStack Table with role/status display |
| Toast notifications | Sonner toast for async feedback |
| Dropdown menus | Radix UI dropdown for context actions |
| Tabs | Radix UI tabs for workbench panels (BK-147) |

---

## 6. Third-Party Integrations

| Service | Purpose | Package | Status | Features using it |
|---------|---------|---------|--------|-------------------|
| Supabase Auth | Passwordless auth (OTP/magic-link) + password auth | `@supabase/supabase-js`, `@supabase/ssr` | Active | FEAT-001, FEAT-002 |
| Supabase PostgreSQL | Primary data store + RLS tenant isolation | `@supabase/supabase-js` | Active | All features |
| Atlassian Jira Cloud | One-way import of User Stories + ACs | Custom HTTP client (`lib/jira/client.ts`) | Active | FEAT-023 |
| Vercel | Hosting (SSR + API routes) + `after()` background execution | Platform (no package) | Active | All; FEAT-023 (after()) |
| Scalar | Interactive API reference viewer | `@scalar/api-reference-react` | Active | FEAT-024 |
| Resend | Email delivery (magic-link, invite emails) | (`RESEND_API_KEY` in `.env.example`) | Unconfirmed — see gaps | FEAT-001 |
| dnd-kit | Drag-and-drop for module tree + test chain reorder | `@dnd-kit/core`, `@dnd-kit/sortable` | Active | FEAT-011, FEAT-020 |
| Monaco Editor | Rich step/assertion content editing | `@monaco-editor/react` | Active | FEAT-014, FEAT-015 |
| TanStack Table | Tabular data (members, ATCs) | `@tanstack/react-table` | Active | FEAT-008 |
| Radix UI | Accessible UI primitives (dialog, dropdown, tabs, tooltip) | `@radix-ui/react-*` | Active | All UI features |
| Sonner | Toast notifications for async feedback | `sonner` | Active | All UI features |

---

## 7. Feature Flags and WIP

### Environment Variables / Feature Flags

| Flag / Env Var | Description | Default | Status |
|----------------|-------------|---------|--------|
| `ATLASSIAN_URL`, `ATLASSIAN_EMAIL`, `ATLASSIAN_API_TOKEN` | Jira integration credentials | Required | Active |
| `NEXT_PUBLIC_SUPABASE_URL`, `NEXT_PUBLIC_SUPABASE_ANON_KEY` | Supabase public config | Required | Active |
| `SUPABASE_SERVICE_ROLE_KEY` | Admin operations (PAT, import worker) | Required | Active |
| `RESEND_API_KEY` | Email delivery (Resend) | Optional | Unconfirmed usage in app code |
| `SUPABASE_JWT_SECRET` | Optional — used in PAT flow | Optional | Active (optional) |

> No `FEATURE_*` / `ENABLE_*` / `BETA_*` prefixed flags found in codebase. Feature gating is at DB level via `feature_flags` table and `plan` column on `workspaces`.

### DB-Level Feature Flags

| Entity | Description | Evidence |
|--------|-------------|---------|
| `feature_flags` table | Workspace-scoped gradual rollout gate | `supabase/migrations/0009_cross_cutting.sql` |
| `workspaces.plan` | Plan tier (`community` / `cloud` / `enterprise`) | Schema only — no app-layer enforcement found |

### Planned / WIP Features

| Feature | Evidence | Status |
|---------|----------|--------|
| Run step result recording + verdict | BK-35/39 mentioned in migrations; step update route not confirmed in file scan | Likely implemented but not found in scan |
| Rate limiting on `check-email` endpoint | ADR-0007 explicitly states "app-level rate limiter not yet shipped" | Planned — follow-up in ADR-0007 |
| `idempotency_keys` expiry cleanup | 24h TTL with no `pg_cron` or cleanup job found | Planned — gap noted in business-data-map |
| Plan tier enforcement (billing gates) | `plan` column on `workspaces`; no feature-gate code found in `app/` | Unimplemented — schema only |
| Jira webhook inbound | No `/webhooks/` route found | Not planned (pull-only architecture) |

---

## 8. QA Relevance

### Feature Test Coverage Matrix

| Feature ID | Feature | Unit | Integration | E2E | Priority | Status |
|------------|---------|------|-------------|-----|----------|--------|
| FEAT-001 | Magic-Link Auth | ❌ | ❌ | ❌ | P0 | Needs E2E |
| FEAT-002 | Password Auth | ❌ | ❌ | ❌ | P0 | Needs E2E (Beta) |
| FEAT-003 | PAT Issuance + Bearer Auth | ⚠️ partial | ❌ | ❌ | P0 | Needs API integration |
| FEAT-004 | PAT Revocation | ❌ | ❌ | ❌ | P1 | Needs API test |
| FEAT-005 | User Profile | ❌ | ❌ | ❌ | P1 | Needs API test |
| FEAT-006 | Workspace CRUD | ❌ | ❌ | ❌ | P0 | Needs E2E + API |
| FEAT-007 | Workspace Invite System | ❌ | ❌ | ❌ | P1 | Needs E2E |
| FEAT-008 | Members Management | ❌ | ❌ | ❌ | P1 | Needs E2E |
| FEAT-009 | Project Management | ❌ | ❌ | ❌ | P1 | Needs E2E |
| FEAT-010 | Environments | ❌ | ❌ | ❌ | P1 | Needs API test |
| FEAT-011 | Module Tree | ⚠️ partial | ❌ | ❌ | P0 | Needs E2E (depth BVA) |
| FEAT-012 | User Story | ❌ | ❌ | ❌ | P1 | Needs API test |
| FEAT-013 | AC Management | ❌ | ❌ | ❌ | P1 | Needs API test |
| FEAT-014 | ATC Creation | ✅ unit tests | ❌ | ❌ | P0 | Needs E2E (anchoring moat) |
| FEAT-015 | ATC Update | ⚠️ partial | ❌ | ❌ | P0 | Needs API test (optimistic lock) |
| FEAT-016 | ATC Duplication | ❌ | ❌ | ❌ | P2 | Needs API test |
| FEAT-017 | ATC Search | ❌ | ❌ | ❌ | P2 | Needs API test |
| FEAT-018 | ATC Usage Report | ❌ | ❌ | ❌ | P2 | Needs API test |
| FEAT-019 | Test Chain Creation | ❌ | ❌ | ❌ | P1 | Needs E2E (idempotency) |
| FEAT-020 | Test Chain Reorder | ❌ | ❌ | ❌ | P1 | Needs E2E (optimistic lock) |
| FEAT-021 | Test Tagging | ❌ | ❌ | ❌ | P2 | Needs API test |
| FEAT-022 | Test Run Execution | ❌ | ❌ | ❌ | P0 | Needs E2E (snapshot, verdict gap) |
| FEAT-023 | Jira Import | ❌ | ❌ | ❌ | P1 | Needs API test (async job) |
| FEAT-024 | OpenAPI Docs | ❌ | ❌ | ❌ | P3 | Smoke test only |
| FEAT-025 | Health Check | ❌ | ❌ | ❌ | P3 | Smoke test |

Legend: ✅ Covered, ⚠️ Partial, ❌ Not covered

> **Note on unit tests**: `lib/` has 19 `.test.ts` files covering validation, builder-guards, optimistic-lock, sanitize, markdown, modules, ADF/AC extraction, import runner, RLS parity and isolation. These are internal unit tests on the target repo — not QA E2E/integration tests.

### High-Risk Features (Prioritize Testing)

| Feature | Risk | Reason |
|---------|------|--------|
| FEAT-014 — ATC Creation (anchoring moat) | HIGH | Core invariant — orphan ATC must be impossible; edge cases on AC linkage validation are security-of-quality boundary |
| FEAT-022 — Test Run Execution | HIGH | Snapshot isolation is critical — ATC edits after run start must NOT affect in-flight run; verdict transitions gate release decisions |
| FEAT-001 / FEAT-002 — Auth | HIGH | Authentication failure = complete product unavailability; security boundary |
| FEAT-003 — PAT + Bearer | HIGH | Bearer auth is the CI/AI pipeline integration point; scope leakage = unauthorized writes |
| FEAT-011 — Module Tree (depth 6) | MEDIUM | Depth enforcement is a structural invariant; break it and the tree becomes unbounded |
| FEAT-023 — Jira Import | MEDIUM | Async job with idempotency; failure modes (Jira auth error, partial run, concurrent imports) require careful state transition testing |
| FEAT-007 — Workspace Invite | MEDIUM | Token security model mirrors PAT (hash-only storage, 7-day expiry) — acceptance flow has time-based edge cases |
| FEAT-019 — Test Chain (idempotency) | MEDIUM | Idempotency-Key header is required; replay semantics must be verified to prevent duplicate chains on network retry |

---

## 9. Discovery Gaps

| Gap | Where Looked | Impact on Testing |
|-----|-------------|------------------|
| **Run step result + verdict route** | `app/api/v1/runs/[id]/route.ts` — only GET found; no PATCH steps/{stepId} or POST verdict route | Cannot write E2E for run completion lifecycle without confirming these routes exist; BK-35/39 referenced in migrations suggests shipped but not found in file scan |
| **Project PATCH (rename) + DELETE** | `app/api/v1/workspaces/[id]/projects/route.ts` — only POST; no `/projects/{id}` route file | Cannot test project rename/delete; unclear if feature exists or is planned |
| **Test chain DELETE** | No `DELETE /api/v1/tests/{id}` route file found | Cannot test test chain removal; unclear if soft-delete exists |
| **ATC soft-archive/delete** | No `DELETE /api/v1/atcs/{id}` route file found | ATC status transitions via Runs are confirmed; but direct archive unclear |
| **Plan tier enforcement** | `workspaces.plan` column + `feature_flags` table — no app-layer gate code found | Cannot test `cloud`/`enterprise` feature restrictions; billing boundary is invisible |
| **Email delivery provider** | `RESEND_API_KEY` in `.env.example`; no Resend import in app code | Magic-link + invite email testing requires confirming Supabase dashboard SMTP config vs Resend |
| **`idempotency_keys` cleanup job** | No `pg_cron`, no cleanup scripts | Expired key rows accumulate; long-term pollution of `idempotency_keys` table |
| **Rate limiting on `check-email`** | ADR-0007 explicitly open | `check-email` reads `auth.users` with no GoTrue throttling; rate limiter not yet shipped — load testing boundary unknown |
| **Workspace member list API** | No `GET /api/v1/workspaces/{id}/members` route found | Members page likely loads via a different mechanism (RLS-scoped query from UI) — API contract for member list unclear |
| **Activity log event catalog** | `activity_log` table in DB; events referenced (`atc.created`, `test.reordered`, `module.archived`) | Full event type catalog not confirmed — cannot write event-audit tests without knowing all event codes |
