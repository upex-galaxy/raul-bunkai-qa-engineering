# Business Data Map — Bunkai TMS

```
+===========================================================+
|                      BUNKAI TMS                           |
|   Test Management System for QA-centric teams             |
|   Enforced AC-to-ATC traceability · Jira import           |
|   REST API · PAT auth · Workspace RBAC                    |
|   Generated: 2026-06-24                                   |
+===========================================================+
```

---

## Executive Summary

Bunkai TMS solves a structural gap that plagues every team running Jira-centric workflows: test cases live in disconnected spreadsheets while the User Stories and Acceptance Criteria they are supposed to verify sit in Jira. Teams lose traceability the moment an engineer forgets to link a test to the story it covers. Over time, the test suite becomes a collection of orphaned cases — test cases that cover nothing verifiable.

Bunkai enforces a structural invariant called the **anchoring moat**: every Acceptance Test Case (ATC) must reference at least one Acceptance Criterion before it can be saved. The system makes accidental orphans impossible. Teams import User Stories directly from Jira (via JQL), bring the associated Acceptance Criteria in, and then author ATCs anchored to those criteria. The result is a traceable chain: `UserStory → AcceptanceCriterion → ATC`, visible in a mind-map view and queryable by tags or full-text search.

Beyond authoring, Bunkai supports **Test chains** — ordered sequences of ATCs that model full end-to-end execution flows — and a **Runs** subsystem that captures execution snapshots per environment. A REST API with PAT-based authentication enables AI agents and CI pipelines to create and execute ATCs programmatically. The workspace RBAC model (viewer / member / admin / owner) and plan tiers (community / cloud / enterprise) position the product for multi-team, multi-tenant deployment.

```
+---------------------+     +-------------------+     +------------------+
|   QA Engineer /     |     |                   |     |  Jira Cloud      |
|   QA Lead           |     |   BUNKAI TMS      |     |  (one-way import)|
|                     +---->+   Next.js 15      +---->+  User Stories    |
|   Manages ATCs,     |     |   Supabase Pg 16  |     |  Acceptance      |
|   Test Plans, Runs  |     |   REST API v1     |     |  Criteria (ADF)  |
+---------------------+     +--------+----------+     +------------------+
                                      |
+---------------------+               |               +------------------+
|   AI Agent / CLI    |               |               |  Supabase Auth   |
|                     +-------------->+               |  (magic-link,    |
|   Programmatic ATC  |  Bearer PAT   |               |   session JWTs)  |
|   create + run      |               +-------------->+                  |
+---------------------+               |               +------------------+
                                       |
                             +---------+---------+
                             |  Vercel Serverless |
                             |  Edge Middleware   |
                             |  after() worker    |
                             +-------------------+
```

---

## Entity Map

```
                         +----------------+
                         |   Workspace    |  (tenant boundary)
                         +-------+--------+
                                 |
          +----------------------+-------------------------+
          |                      |                         |
    +-----+-----+         +------+------+         +--------+-------+
    |  Project  |         |  Team/      |         | Access Token   |
    | (app under|         |  Members    |         | (PAT)          |
    |   test)   |         +-------------+         +----------------+
    +-----+-----+
          |
    +-----+------+
    |   Module   |  (tree, max depth 6)
    +-----+------+
          |
    +-----+-------+
    | User Story  |<---- external_id (Jira issue key)
    +-----+-------+
          |
   +------+--------+
   | Acceptance    |
   | Criterion     |  (AC)
   +------+--------+
          | M:N (anchoring moat)
   +------+--------+
   |   ATC         |  (Acceptance Test Case)
   +------+--------+
          |               +-------------+
     +----+-----+         |  Test       |  (ordered ATC chain)
     | Steps    |         +------+------+
     | Assertions|                |
     +-----------+         +------+------+
                           | TestStep    |
                           +------+------+
                                  |
                           +------+------+
                           |  Run        |  (execution snapshot)
                           +-------------+
                                  |
                        +---------+---------+
                        | RunATC  | RunStep |
                        +---------+---------+
```

| Entity | Business Role | Why It Exists |
|--------|--------------|---------------|
| `Workspace` | Top-level tenant boundary | Isolates all data per organization; enforces multi-tenancy via RLS |
| `WorkspaceMember` | RBAC join table | One user can have different roles in different workspaces |
| `WorkspaceInvite` | Invite system | Allows admin+ to onboard users without pre-existing accounts |
| `Project` | Application Under Test (AUT) | Scopes the module tree and ATCs to one testable product |
| `Module` | Hierarchical test folder (max depth 6) | Organizes stories/ATCs the same way an app organizes features |
| `UserStory` | Business intent unit | Links Jira stories into the TMS; anchors ACs and ATCs |
| `AcceptanceCriterion` (AC) | Verifiable story condition | The minimum provable contract for a story; the anchor for ATCs |
| `ATC` | Acceptance Test Case | Atomic, traceable test case; must link ≥1 AC (the anchoring moat) |
| `ATCStep` | Ordered action in an ATC | Captures the procedural "how to" for manual or automated execution |
| `ATCAssertion` | Post-condition check in an ATC | Declares what must be true after the steps complete |
| `Test` | Ordered ATC chain | Models a full end-to-end execution sequence from multiple atomic cases |
| `TestStep` | ATC reference within a Test | Holds the position of each ATC in the chain |
| `Run` | Execution snapshot | Captures a point-in-time execution of a Test against an environment |
| `ProjectEnvironment` | Target environment for a Run | Decouples the test definition (Test) from the target (Staging/Production) |
| `AccessToken` (PAT) | Machine API key | Enables CLI/AI-agent access with scoped permissions |
| `ImportJob` | Async Jira import task | Tracks the lifecycle of a JQL-driven batch import of stories + ACs |
| `ActivityLog` | Audit trail | Workspace-scoped event log for compliance and debugging |
| `FeatureFlag` | Gradual rollout gate | Enables/disables features per workspace without a deploy |
| `IdempotencyKey` | POST replay guard | Prevents duplicate Test chain creation on network retry |

---

## Business Flows

### Flow 1 — User Authentication (Browser Session)

```
  User                Edge Middleware         Next.js          Supabase Auth
   |                        |                    |                   |
   |-- GET /projects ------->|                   |                   |
   |                        |-- getUser() ------>|                   |
   |                        |                    |<-- null session --|
   |<-- 302 /login?next=... |                    |                   |
   |                        |                    |                   |
   |-- POST /api/v1/auth/magic-link (email) ---->|                   |
   |                        |                    |-- signInWithOtp ->|
   |                        |                    |<-- 200 sent ------|
   |                        |                    |                   |
   |-- (clicks email link) -- GET /auth?code=... |                   |
   |                        |                    |-- exchangeCode -->|
   |                        |                    |<-- session JWT ---|
   |<-- Set-Cookie; 302 /projects               |                   |
   |                        |                    |                   |
   |-- GET /projects ------->|                   |                   |
   |                        |-- getUser() valid ->                   |
   |<-- 200 projects page --------------------------------           |
```

**Steps:**
1. User hits a protected route (`/projects`, `/onboarding`) with no session cookie.
2. Edge Middleware calls `getUser()` — null session → 302 redirect to `/login?next=<path>`.
3. User submits email → `POST /api/v1/auth/magic-link` → Supabase `signInWithOtp`.
4. Supabase sends magic-link email (OTP code).
5. User clicks link → `GET /auth?code=...` → `exchangeCodeForSession(code)`.
6. Session JWT + refresh token set as cookie; redirect to `/projects` (or `?next=`).
7. On subsequent requests, Edge Middleware validates JWT and refreshes expiring sessions.

**Business Rules:**
- Already-authenticated users hitting `/login` are redirected to `/projects`.
- User with no workspace after login is redirected from `/projects` to `/onboarding`.
- Middleware runs on EVERY request to protected prefixes — no RLS bypass possible.

**Code Paths:** `middleware.ts`, `app/(auth)/login/`, `app/api/v1/auth/magic-link/route.ts`, `lib/supabase/server.ts`

Source: `SRS/functional-specs.md FR-001`

---

### Flow 2 — PAT Issuance and Bearer Authentication

```
  API Client              Next.js API           DB (access_tokens)
      |                       |                        |
      |-- POST /api/v1/tokens |                        |
      |   (name, scopes,      |                        |
      |    workspace_id) ----->|                       |
      |                       |-- mint bk_pat_<12>.<secret>
      |                       |-- INSERT access_tokens (prefix, scopes)
      |                       |-- INSERT access_token_secrets (SHA-256)
      |<-- 201 { token: "bk_pat_..." }                 |
      |                       |                        |
      |-- POST /api/v1/atcs --|                        |
      |   Authorization: Bearer bk_pat_...             |
      |                       |-- parse prefix -------->|
      |                       |<-- candidate rows ------|
      |                       |-- fetch SHA-256 hash ---|
      |                       |-- SHA-256(fullSecret) compare
      |                       |-- check revoked_at, expires_at
      |                       |-- mint short-lived user JWT
      |<-- 201 ATC created ---|-- RLS as auth.uid() ---|
```

**Steps:**
1. Authenticated user calls `POST /api/v1/tokens` with name + scopes.
2. System mints `bk_pat_<12-char-prefix>.<base64url-secret>`.
3. Stores `token_prefix` in `access_tokens`; SHA-256 hash in `access_token_secrets` (separate table for security).
4. Returns full raw token in 201 response — **only time it is readable**.
5. Subsequent API calls present `Authorization: Bearer bk_pat_...`.
6. Bearer middleware: lookup by prefix → fetch hash → compare SHA-256 → check revocation/expiry.
7. On success: mints short-lived user JWT, creates impersonating Supabase client — RLS applies normally.

**Business Rules:**
- Raw PAT secret stored nowhere — only SHA-256 hash.
- All bearer auth failures produce uniform 401 (no hint which check failed — INV-2).
- Revocation is soft (`revoked_at` timestamp) — no DELETE on `access_tokens`.
- PAT carries only declared scopes; cookie session carries ALL capabilities.

**Code Paths:** `app/api/v1/tokens/route.ts`, `lib/api/middleware/bearer.ts`, `lib/api/principal.ts`, `supabase/migrations/0008_access_tokens.sql`, `supabase/migrations/0011_split_token_secrets.sql`

Source: `SRS/functional-specs.md FR-003`

---

### Flow 3 — Workspace and Project Setup (Onboarding)

```
  New User            Next.js          bunkai_bootstrap_workspace (RPC)
      |                   |                          |
      |-- POST onboarding form (slug, name) -------->|
      |                   |-- BEGIN tx               |
      |                   |   INSERT workspaces (slug, name, plan='community')
      |                   |   INSERT workspace_members (role='owner', status='active')
      |                   |-- COMMIT                 |
      |<-- 302 /projects  |                          |
      |                   |                          |
      |-- POST /api/v1/projects (name, slug) ------->|
      |                   |-- INSERT projects (workspace_id)
      |<-- 201 project    |                          |
```

**Steps:**
1. First-time user redirected to `/onboarding` after login (no workspace exists).
2. User fills in workspace name + slug → `bunkai_bootstrap_workspace` RPC (SECURITY DEFINER).
3. RPC atomically inserts `workspaces` + `workspace_members` (owner, active) in one transaction.
4. Workspace `plan` defaults to `community`; `slug` must be lowercase, digits, hyphens, 3–40 chars.
5. User creates their first `Project` inside the workspace.

**Business Rules:**
- Workspace slug must be globally unique (URL-safe); slug collision → 23505 surfaced to user.
- Owner role is automatically assigned on workspace creation.
- The `bunkai_bootstrap_workspace` RPC bypasses RLS internally to solve the chicken-and-egg problem (INSERT to workspace + member simultaneously).

**Code Paths:** `app/(app)/onboarding/`, `supabase/migrations/0006_bootstrap_workspace.sql`, `app/(app)/projects/create-project-form.tsx`

---

### Flow 4 — ATC Creation with Enforced AC Linkage

```
  QA Engineer         POST /api/v1/atcs        bunkai_save_atc RPC
       |                     |                         |
       |-- POST (title, layer, steps,                  |
       |   assertions, acceptance_criterion_ids,        |
       |   module_id, user_story_id) -------->          |
       |                     |-- Zod parse (422 on fail)
       |                     |-- stepPositionsError() (422 steps_position_invalid)
       |                     |-- sanitize steps/assertions
       |                     |                         |
       |                     |-- bunkai_save_atc ------>|
       |                     |                         |-- derive project_id from user_story_id
       |                     |                         |-- validate module_id IN project
       |                     |                         |-- validate AC IDs IN user_story_id
       |                     |                         |-- compute slug (immutable)
       |                     |                         |-- INSERT atcs + atc_steps
       |                     |                         |-- INSERT atc_assertions
       |                     |                         |-- INSERT atc_acceptance_criteria
       |                     |                         |-- emit activity_log (atc.created)
       |                     |<-- SECURITY DEFINER returns ATC
       |<-- 201 { atc: {...} }
```

**Steps:**
1. Member+ user submits ATC form or `POST /api/v1/atcs` (Bearer with `atc:write` scope).
2. `AtcCreateBodySchema.parse()` validates title (3–200 chars), layer, tags (max 10), steps (min 1), content budget (2048 bytes/field).
3. `stepPositionsError()` checks positions are strictly increasing integers starting at 1.
4. `sanitizeAtcSteps` / `sanitizeAtcAssertions` clean content.
5. `bunkai_save_atc` SECURITY DEFINER RPC validates cross-entity integrity and writes all tables transactionally.
6. Activity event `atc.created` emitted to `activity_log`.

**Business Rules (BR-ATC-001 through BR-ATC-005):**
- `acceptance_criterion_ids` must contain ≥1 UUID (the anchoring moat) — no ATC without an AC link.
- ATC slug computed from title at creation and is immutable.
- Step positions: integers, strictly increasing from 1 (gaps allowed, e.g. [1, 2, 5] is valid).
- ATC status defaults to `unrun`.
- Content fields: max 2048 bytes UTF-8 (byte count, not char count).

**Code Paths:** `app/api/v1/atcs/route.ts`, `lib/atcs/validation.ts`, `lib/atcs/builder-guards.ts`, `lib/supabase/rpc.ts`, `supabase/migrations/0004_atcs.sql`, `supabase/migrations/0007_save_atc.sql`

Source: `SRS/functional-specs.md FR-002`

---

### Flow 5 — Module Tree Management

```
  QA Engineer        Module CRUD API         bunkai_move_module RPC
       |                    |                          |
       |-- POST /module (name, parent_id) ----------->|
       |                    |-- CHECK path depth <= 6 |
       |                    |-- INSERT modules (path, position)
       |<-- 201 module      |                          |
       |                    |                          |
       |-- PATCH /module (move to new parent) -------->|
       |                    |                         |-- check target depth + subtree depth
       |                    |                         |-- if > 6: raise 45002 depth_exceeded
       |                    |                         |-- UPDATE path for subtree (cascade)
       |<-- 200 OK or 422 depth_exceeded              |
       |                    |                          |
       |-- DELETE /module   |                          |
       |                    |-- SET archived_at = now()   (soft delete)
       |<-- 200 OK          |                          |
```

**Steps:**
1. Member+ user creates a module under a parent (or at root of a project).
2. Path is a slash-separated string; depth = slash count + 1; max is 6 levels.
3. DB CHECK constraint enforces `char_length(path) - char_length(replace(path, '/', '')) <= 5`.
4. Moving a module uses the `bunkai_move_module` RPC, which checks the entire subtree depth after move.
5. Deletion is soft (`archived_at` timestamp); soft-deleted modules are hidden from active tree queries.

**Business Rules (BR-MOD-001 through BR-MOD-003):**
- Max depth 6 enforced at both DB (CHECK) and RPC layers.
- Move that would push any descendant beyond depth 6 → error `depth_exceeded` (pg 45002).
- Soft-deleted modules return `archived_at IS NULL` = false; filtered from all active tree queries.

**Code Paths:** `supabase/migrations/0002_projects_modules.sql`, `supabase/migrations/0014_module_soft_delete.sql`, `supabase/migrations/0015_module_move.sql`, `supabase/migrations/0023_module_activity_log.sql`

Source: `SRS/functional-specs.md FR-007`

---

### Flow 6 — Jira Import (Async JQL-based)

```
  QA Engineer       POST /api/v1/imports     Vercel after()        Jira REST API v3
       |                    |                     |                      |
       |-- POST (project_id, jql) ------------->  |                      |
       |                    |-- check active import (409 if exists)      |
       |                    |-- INSERT import_jobs (status='queued')     |
       |                    |-- after(() => runImportJob(id)) ---------->|
       |<-- 202 { job: { id, status: 'queued' } }                        |
       |                    |                     |                      |
       |                    |          (async, after HTTP response)       |
       |                    |                     |-- UPDATE status='running'
       |                    |                     |-- searchIssues(jql) ->|
       |                    |                     |<-- page of issues ----|
       |                    |                     |-- per issue:          |
       |                    |                     |   ADF->Markdown       |
       |                    |                     |   extract ACs         |
       |                    |                     |   UPSERT user_stories |
       |                    |                     |   UPSERT acceptance_criteria
       |                    |                     |-- UPDATE status='completed'
```

**Steps:**
1. Member+ user opens the Jira import dialog, enters a JQL query.
2. `POST /api/v1/imports` validates `jql` (trim, 1–2000 chars) and `project_id`.
3. Partial unique index ensures only one `queued` or `running` job per project → 409 if active.
4. Import job row inserted with `status='queued'`; `after(() => runImportJob(jobId))` schedules background work.
5. Response returns 202 immediately — UI polls for job status.
6. Background worker: atomically claims job (`queued → running`), pages through Jira (PAGE_SIZE=100, MAX_PAGES=1000).
7. Per issue: converts ADF description to Markdown (truncated at 50 KB), extracts ACs, routes to Module (component-name or auto-created "Inbox"), UPSERTs `user_stories` keyed on `external_id` (Jira key), UPSERTs `acceptance_criteria`.
8. On completion: `status='completed'` + counters; on Jira auth error: `status='failed'`.

**Business Rules (BR-IMP-001 through BR-IMP-004):**
- At most one active import per project (partial unique index).
- Import is idempotent: re-running same JQL updates (not duplicates) existing stories by `external_id`.
- Jira auth error fails the entire job; per-issue errors appended to `errors[]` JSONB array.
- Description truncated to 50 KB with marker (prevents unbounded DB rows).

**Code Paths:** `app/api/v1/imports/route.ts`, `lib/jira/import-runner.ts`, `lib/jira/client.ts`, `lib/jira/extract-acceptance-criteria.ts`, `lib/jira/adf-to-markdown.ts`, `supabase/migrations/0019_import_jobs.sql`, `supabase/migrations/0020_import_jobs_one_active.sql`

Source: `SRS/functional-specs.md FR-004`

---

### Flow 7 — Test Chain Creation

```
  QA Engineer         POST /api/v1/tests       bunkai_create_test RPC
       |                      |                          |
       |-- POST (title, atc_ids, workspace_id)            |
       |   Idempotency-Key: <key> ------------------>    |
       |                      |-- beginIdempotentRequest |
       |                      |-- check key not seen before (400 if missing/invalid)
       |                      |                          |
       |                      |-- bunkai_create_test ---->|
       |                      |                          |-- role gate: member+
       |                      |                          |-- chain must have >=1 ATC
       |                      |                          |-- resolve all ATC IDs in workspace
       |                      |                          |   (INV-3: missing = same error as foreign)
       |                      |                          |-- INSERT tests
       |                      |                          |-- INSERT test_steps (ordered)
       |                      |<-- test record -----------|
       |                      |-- recordIdempotencyResult|
       |<-- 201 { test: {...} }
```

**Steps:**
1. Member+ user submits title + ordered list of ATC UUIDs via UI or `POST /api/v1/tests`.
2. `Idempotency-Key` header is required (`[\w-]{8,128}`) — prevents duplicate chains on retry.
3. `bunkai_create_test` SECURITY DEFINER RPC: validates member+ role, chain ≥1 ATC, all ATC IDs within workspace.
4. Cross-workspace or nonexistent ATCs produce the same `atc_not_in_workspace` error (INV-3 non-disclosure).
5. `tests` + `test_steps` inserted transactionally; activity event emitted.
6. Idempotency result stored; replayed on same key+payload.

**Business Rules (BR-TEST-001 through BR-TEST-004):**
- Empty chain → `chain_empty` (422).
- Cross-workspace ATC = nonexistent ATC (same error — non-disclosure of resource existence).
- `Idempotency-Key` header required; missing → 400 `idempotency_key_required`.
- Same ATC may appear multiple times in a chain (no UNIQUE on `(test_id, atc_id)`).

**Code Paths:** `app/api/v1/tests/route.ts`, `lib/api/idempotency.ts`, `supabase/migrations/0024_tests.sql`, `supabase/migrations/0025_test_read.sql`

Source: `SRS/functional-specs.md FR-005`

---

### Flow 8 — Test Run Execution

```
  QA Engineer / Agent    POST /api/v1/runs      bunkai_create_run RPC
         |                      |                         |
         |-- POST (test_id,      |                         |
         |   environment_id,     |                         |
         |   executor_mode)      |                         |
         |   start_token ------->|                         |
         |                      |-- bunkai_create_run ---->|
         |                      |                         |-- role gate: member+
         |                      |                         |-- validate test_id in workspace
         |                      |                         |-- validate environment_id for project
         |                      |                         |-- snapshot: INSERT runs (status='running')
         |                      |                         |-- snapshot: INSERT run_atcs (positions)
         |                      |                         |-- snapshot: INSERT run_steps (step data)
         |                      |<-- run record -----------|
         |<-- 201 { run: {...} } |                         |
         |                      |                         |
         |-- PATCH /runs/:id/steps/:stepId (result) ------>|
         |<-- 200 step updated   |                         |
         |                      |                         |
         |-- POST /runs/:id/verdict (passed|failed|aborted)|
         |<-- 200 run closed     |                         |
```

**Steps:**
1. Member+ user (or agent with `run:execute` scope) starts a Run against a Test + environment.
2. `bunkai_create_run` SECURITY DEFINER RPC: validates membership, checks environment belongs to the Test's project, takes a **snapshot** of all ATCs and steps at start time.
3. Snapshot tables (`run_atcs`, `run_steps`) freeze the Test definition — future ATC edits do not affect the run in progress.
4. Run status starts as `running`; transitions to `passed`, `failed`, or `aborted` on verdict.
5. Individual step results are recorded as the engineer executes each step.

**Business Rules:**
- Executor modes: `human`, `agent`, `ci` (CHECK constraint).
- Environment must be configured for the Test's project (error 45201 `environment_invalid`).
- Test with zero executable steps → error 45202 `no_executable_steps`.
- Run is a snapshot — ATC edits after run start have no retroactive effect.

**Code Paths:** `supabase/migrations/0031_runs.sql`, `supabase/migrations/0032_project_environments_crud.sql`, `app/api/v1/` (runs routes)

---

### Flow 9 — Workspace Invite System

```
  Admin               POST /api/v1/invites      POST /api/v1/invites/accept
     |                       |                          |
     |-- POST (email, role,   |                          |
     |   workspace_id) ------>|                          |
     |                       |-- INSERT workspace_invites|
     |                       |   token_hash (raw returned once)
     |                       |   expires_at = now() + 7d|
     |<-- 201 { token: "..." }|                          |
     |                       |                          |
     |-- (sends email with raw token to invitee)         |
     |                       |                          |
  Invitee                    |                          |
     |-- POST /invites/accept (token) ----------------->|
     |                       |                          |-- hash token, lookup invite
     |                       |                          |-- check expires_at > now()
     |                       |                          |-- check revoked_at IS NULL
     |                       |                          |-- INSERT workspace_members
     |                       |                          |   (role from invite, status='active')
     |                       |                          |-- SET accepted_at = now()
     |<-- 200 membership created                        |
```

**Steps:**
1. Admin+ sends invite: `POST /api/v1/invites` with email, role (viewer/member/admin — owner not invitable), workspace_id.
2. System generates a raw token; stores only the hash; returns raw token once in response.
3. Admin delivers raw token to invitee (email/chat — app does not send invite email directly; Supabase Auth or Resend handles it — see Discovery Gaps).
4. Invitee submits raw token to `POST /api/v1/invites/accept`.
5. System hashes token, finds invite row, checks expiry/revocation, creates `workspace_members` (active), marks invite `accepted_at`.

**Business Rules (BR-INV-001 through BR-INV-004):**
- Invite expires after 7 days.
- Token raw value returned once; only hash stored.
- Admin cannot invite with `owner` role.
- Accepting invite creates an active membership record.

**Code Paths:** `app/api/v1/invites/`, `supabase/migrations/0010_workspace_invites.sql`, `supabase/migrations/0022_invite_integrity_user_lookup.sql`

Source: `SRS/functional-specs.md FR-006`

---

### Flow 10 — Acceptance Criteria Management

```
  QA Engineer         Module/Story tree           DB
       |                     |                    |
       |-- POST /user-stories (title, module_id) ->|
       |<-- 201 story         |                    |
       |                      |                    |
       |-- POST /acceptance-criteria               |
       |   (user_story_id, title, position) ------>|
       |<-- 201 criterion     |                    |
       |                      |                    |
       |-- PATCH /acceptance-criteria/:id (reorder)|
       |<-- 200 updated        |                   |
       |                       |                   |
       |-- DELETE /acceptance-criteria/:id          |
       |                       |-- SET archived_at  |
       |<-- 200 soft deleted   |                    |
```

**Steps:**
1. Member+ user creates a User Story inside a Module.
2. User Story can optionally link to a Jira issue via `external_id` + `external_url`.
3. Acceptance Criteria are added to the story with a `position` (sortable).
4. ACs can be reordered (migration `0017_acceptance_criteria_ordering.sql`).
5. ACs are soft-deleted (`archived_at`); archived ACs cannot be referenced by new ATCs.

**Business Rules:**
- User Story `external_id` uniqueness enforced per module (`supabase/migrations/0016_user_story_uniqueness.sql`).
- AC position is an ordered integer; the system reorders on AC CRUD (migration 0017).
- Once an ATC references an AC, the AC cannot be hard-deleted (ON DELETE RESTRICT on `atc_acceptance_criteria`).

**Code Paths:** `supabase/migrations/0003_authoring.sql`, `supabase/migrations/0016_user_story_uniqueness.sql`, `supabase/migrations/0017_acceptance_criteria_ordering.sql`

---

### Flow 11 — ATC Search and Discovery

```
  QA Engineer / Agent    GET /api/v1/atcs?q=<term>&tags=...
         |                       |
         |-- query (text, tags, layer, project_id) ----------->|
         |                       |-- bunkai_search_atcs RPC     |
         |                       |   (tsvector GIN index)       |
         |                       |-- filter by workspace RLS    |
         |<-- 200 { atcs: [...] } |                             |
```

**Steps:**
1. User or agent queries ATCs by free text and/or tags.
2. Full-text search uses the `atcs.tsv` tsvector column with a GIN index.
3. `bunkai_search_atcs` RPC handles the search — workspace RLS applies.
4. Results can be filtered by layer (UI/API/Unit), tags, module, or user story.

**Code Paths:** `supabase/migrations/0027_atc_search.sql`, `app/api/v1/atcs/` (GET handler)

---

### Flow 12 — ATC Duplication

```
  QA Engineer     POST /api/v1/atcs/:id/duplicate      bunkai_duplicate_atc RPC
       |                        |                              |
       |-- POST (target module_id, user_story_id) ----------->|
       |                        |                             |-- copy atc row
       |                        |                             |-- copy atc_steps
       |                        |                             |-- copy atc_assertions
       |                        |                             |-- copy atc_acceptance_criteria
       |                        |                             |-- new slug (appended -copy-N)
       |<-- 201 { atc: cloned } |                             |
```

**Steps:**
1. Member+ user triggers ATC duplication (from UI context menu or API).
2. `bunkai_duplicate_atc` RPC clones the ATC with all its steps, assertions, and AC links.
3. New slug is generated (title-based + `-copy-N` suffix to avoid collision).
4. Duplicated ATC starts in `unrun` status.

**Code Paths:** `supabase/migrations/0028_atc_duplicate.sql`

---

### Flow 13 — OpenAPI Documentation Serving

```
  Client / AI Agent    GET /api/openapi    GET /api/docs
       |                     |                   |
       |-- GET /api/openapi ->|                   |
       |                     |-- force-static cache (max-age=300)
       |                     |-- serve public/openapi.json
       |<-- 200 OpenAPI 3.1 JSON                  |
       |                     |                   |
       |-- GET /api/docs ---->|                   |
       |                     |-- Scalar viewer  -->|
       |<-- HTML (interactive docs)              |
```

**Steps:**
1. `GET /api/openapi` returns the pre-generated `public/openapi.json` (force-static, 5-minute cache).
2. Spec is generated by `bun run openapi:gen` from Zod schemas registered in `lib/openapi/registry.ts`.
3. `GET /api/docs` serves the Scalar API reference viewer (React component, renders the spec interactively).
4. `GET /api/v1` returns a JSON index of the API version, openapi path, docs path.

**Code Paths:** `app/api/openapi/route.ts`, `app/api/docs/page.tsx`, `lib/openapi/registry.ts`, `scripts/openapi-gen.ts`

---

## State Machines

### ATC Status (`atcs.status`)

```
                +--------+
   [created] -->| unrun  |
                +---+----+
                    |
                    v
                +--------+
                | running |<--------+
                +----+----+         |
                     |              |
          +----------+-----------+  |
          |          |           |  |
          v          v           v  |
       +------+   +------+   +----------+   +----------+
       | pass |   | fail |   | blocked  |   | skipped  |
       +--+---+   +--+---+   +----+-----+   +----+-----+
          |          |            |               |
          +----------+------------+---------------+
                              | (re-run)
                              v
                          running
```

| From | To | Triggering Event | Effects |
|------|----|-----------------|---------|
| `[created]` | `unrun` | ATC saved | Default status; not yet executed |
| `unrun` | `running` | Execution started | Marks test as in-progress |
| `running` | `pass` | Tester marks pass | Positive result recorded |
| `running` | `fail` | Tester marks fail | Negative result; defect implied |
| `running` | `blocked` | Dependency missing | External blocker prevents execution |
| `running` | `skipped` | Out of scope | Intentionally not executed this cycle |
| `pass/fail/blocked/skipped` | `running` | Re-run triggered | ATC returns to active execution state |

**Business Rules:**
- `unrun` is the only entry state — cannot create an ATC already in `pass` or `fail`.
- `running` is the only intermediate state; all terminal states return through `running` on re-execution.
- Status CHECK constraint defined in `supabase/migrations/0004_atcs.sql` line 62.

Source: `domain-glossary.md §6`, `SRS/functional-specs.md §State Machines`

---

### Import Job Status (`import_jobs.status`)

```
                  +--------+
   [enqueued] --> | queued |
                  +---+----+
                      | (after() worker claims — atomic UPDATE)
                      v
                  +--------+
                  | running |
                  +----+----+
                       |
           +-----------+-----------+
           |                       |
           v                       v
      +-----------+          +--------+
      | completed |          | failed |
      +-----------+          +--------+
```

| From | To | Triggering Event | Effects |
|------|----|-----------------|---------|
| `[POST /api/v1/imports]` | `queued` | User enqueues import | Job row created; Vercel after() scheduled |
| `queued` | `running` | Worker atomic claim | `UPDATE WHERE status='queued'` prevents parallel execution |
| `running` | `completed` | All Jira pages processed | Counters (`created/updated/skipped`) set |
| `running` | `failed` | Jira auth error or unrecoverable failure | `errors[]` JSONB populated |

**Business Rules:**
- Partial unique index on `(project_id) WHERE status IN ('queued','running')` — only one active import per project.
- `queued → running` claim is atomic; concurrent triggers are no-ops.
- No retry mechanism in current implementation — `failed` is a terminal state.

Source: `SRS/functional-specs.md §State Machines`, `supabase/migrations/0019_import_jobs.sql`

---

### Workspace Member Status (`workspace_members.status`)

```
              +----------+
   [invite]-->| invited  |  (workspace_invites row created)
              +----+-----+
                   | (invitee accepts)
                   v
              +--------+
              | active |<--------+
              +---+----+         |
                  |              |
                  v              |
              +-----------+      | (admin reinstates)
              | suspended |------+
              +-----------+
```

| From | To | Triggering Event | Effects |
|------|----|-----------------|---------|
| `[invite sent]` | `invited` | Admin+ creates workspace invite | `workspace_members` row with `status='invited'` |
| `invited` | `active` | Invitee accepts token | `accepted_at` set on invite; member can access workspace |
| `active` | `suspended` | Admin suspends member | RLS denies all data access immediately |
| `suspended` | `active` | Admin reinstates | Access restored |

**Business Rules:**
- RLS helper `bunkai_is_workspace_member()` checks `status = 'active'`.
- Invited-but-not-yet-accepted member has NO data access.
- Invite expires after 7 days — if not accepted, `invited` member is effectively frozen.

Source: `domain-glossary.md §6`, `supabase/migrations/0001_tenancy.sql`, `supabase/migrations/0010_workspace_invites.sql`

---

### Workspace Invite Lifecycle (`workspace_invites`)

```
              +---------+
   [created]->| pending |
              +----+----+
                   |
        +----------+-----------+
        |          |           |
        v          v           v
   +----------+ +--------+ +---------+
   | accepted | | revoked| | expired |
   +----------+ +--------+ +---------+
   (accepted_at  (revoked_at  (expires_at
    IS SET)       IS SET)      < now())
```

| From | To | Triggering Event | Effects |
|------|----|-----------------|---------|
| `[admin POST]` | `pending` | Invite created | 7-day expiry set; raw token returned once |
| `pending` | `accepted` | Invitee submits token | `accepted_at` timestamptz set; membership created |
| `pending` | `revoked` | Admin revokes | `revoked_at` timestamptz set; token rejected |
| `pending` | `expired` | `expires_at < now()` | Passive expiry; accept endpoint rejects |

Source: `SRS/functional-specs.md §State Machines`, `supabase/migrations/0010_workspace_invites.sql`

---

### Test Run Status (`runs.status`)

```
              +---------+
   [created]->| running |
              +----+----+
                   |
        +----------+-----------+
        |          |           |
        v          v           v
   +--------+  +--------+  +---------+
   | passed |  | failed |  | aborted |
   +--------+  +--------+  +---------+
```

| From | To | Triggering Event | Effects |
|------|----|-----------------|---------|
| `[bunkai_create_run]` | `running` | Run created with snapshot | `run_atcs` + `run_steps` snapshot frozen |
| `running` | `passed` | Verdict: all steps pass | `finished_at` set; version bumped |
| `running` | `failed` | Verdict: ≥1 step failed | `finished_at` set; version bumped |
| `running` | `aborted` | Run manually aborted | `finished_at` set; run halted mid-execution |

**Business Rules:**
- Run is a snapshot — the frozen `run_atcs`/`run_steps` are never updated retroactively.
- `executor_mode` CHECK: `human`, `agent`, `ci`.
- Optimistic lock (`version` column) for BK-39 concurrent verdict conflict prevention.

Source: `supabase/migrations/0031_runs.sql`

---

### Idempotency Key State (`idempotency_keys.status`)

```
              +---------+
   [POST]  -->| pending |
              +----+----+
                   |
        +----------+----------+
        |                     |
        v                     v
   +-----------+           +--------+
   | succeeded |           | failed |
   +-----------+           +--------+
   (replay ->              (next request
    snapshot)               re-claims key)
```

| From | To | Triggering Event | Effects |
|------|----|-----------------|---------|
| `[beginIdempotentRequest]` | `pending` | First POST with this key | Claim inserted |
| `pending` | `succeeded` | Handler completes successfully | Response snapshot stored; replayed on retry |
| `pending` | `failed` | Handler throws | No snapshot stored; next request with same key re-claims |

Source: `SRS/functional-specs.md §State Machines`, `lib/api/idempotency.ts`

---

## Automatic Processes

### DB Triggers

| Trigger | Table | Event | Purpose |
|---------|-------|-------|---------|
| `runs_set_updated_at` | `public.runs` | BEFORE UPDATE | Auto-updates `updated_at` timestamp; mirrors pattern from 0009_cross_cutting `bunkai_set_updated_at()` |
| `bunkai_set_updated_at` (shared fn) | `public.atcs`, `public.runs`, `feature_flags` | BEFORE UPDATE | Common trigger function; prevents stale `updated_at` values |
| Bootstrap trigger (implicit) | `auth.users` | AFTER INSERT | `0006_bootstrap_workspace.sql` — seeds first workspace for new signups (implicit via Supabase Auth user creation flow) |

> **Note**: Full trigger enumeration requires direct DB introspection (`SELECT trigger_name FROM information_schema.triggers`). The above are verified from migration files. Discovery Gap: additional triggers may exist on older tables (0001–0018) not fully scanned.

### Cron Jobs

| Job | Schedule | Purpose | Evidence |
|-----|---------|---------|---------|
| None confirmed | — | — | No `pg_cron` extension found in migrations; no Vercel Cron configuration found; no scheduled functions in `app/api/` directory |

> **Discovery Gap**: `idempotency_keys` has a 24-hour TTL (`expires_at`). A cleanup cron job is implied but not found in code. Expired rows may accumulate without a cleanup mechanism.

### Incoming Webhooks

| Webhook | Source | Handler | Purpose |
|---------|--------|---------|---------|
| None confirmed | — | — | Jira integration is outbound-only (Bunkai pulls from Jira; Jira does not push to Bunkai). No `/api/v1/webhooks/` route found. |

> **Discovery Gap**: No Jira webhook inbound handler found. Story status changes in Jira do not automatically propagate to Bunkai `user_stories` — import is the only sync mechanism and it is user-triggered.

---

## External Integrations

### Supabase (Auth + Database)

```
  Next.js App                    Supabase Platform
       |                               |
       |-- supabase.auth.signInWithOtp -->  Auth Service
       |<-- magic link email sent ------    (manages OTP + sessions)
       |                               |
       |-- supabase.auth.getUser() ---->    Auth Service
       |<-- user object + session ------    (JWT verification)
       |                               |
       |-- supabase.from('atcs').select -->  PostgREST (RLS enforced)
       |<-- filtered rows ---------------    PostgreSQL 16
       |                               |
       |-- rpc('bunkai_save_atc') ------>    PostgreSQL (SECURITY DEFINER)
       |<-- ATC record -----------------    (bypasses RLS internally)
       |                               |
  Admin Client (SUPABASE_SERVICE_ROLE_KEY)
       |-- admin-only operations ------->    PostgreSQL (RLS bypassed)
       |   (PAT minting, import worker)      Used for: access_token_secrets
       |                               |        import_jobs writes
```

**What it does:** Primary data store (PostgreSQL 16) + passwordless auth (magic-link) + Row-Level Security (tenant isolation).

**How it affects data:** ALL application data lives in Supabase. RLS policies enforce workspace-level isolation — no application-level filtering needed. SECURITY DEFINER RPCs (prefixed `bunkai_*`) handle cross-entity writes atomically.

**Flows depending on it:** All flows. RLS is the authorization backbone; removing Supabase requires replacing the entire auth + DB stack.

**Key env vars:** `NEXT_PUBLIC_SUPABASE_URL`, `NEXT_PUBLIC_SUPABASE_ANON_KEY`, `SUPABASE_SERVICE_ROLE_KEY`, `SUPABASE_JWT_SECRET` (optional, PAT flow)

---

### Atlassian Jira Cloud (One-way Import)

```
  Vercel after() Worker              Jira REST API v3
          |                                |
          |-- GET /rest/api/3/search       |
          |   (JQL, startAt, maxResults) ->|
          |<-- issues[] (JSON, ADF) -------+
          |                                |
          |-- (per issue)                  |
          |   convert ADF -> Markdown      |
          |   extract Acceptance Criteria  |
          |   UPSERT user_stories          |
          |   UPSERT acceptance_criteria   |
          |                                |
          |-- GET /rest/api/3/project ----->|  (used for module routing)
          |<-- project components[] --------+
```

**What it does:** One-way source of User Stories and Acceptance Criteria. Bunkai pulls; Jira does not push.

**How it affects data:** Creates/updates `user_stories` and `acceptance_criteria` in Bunkai. Import is idempotent (keyed on `external_id` = Jira issue key). ADF (Atlassian Document Format) descriptions are converted to Markdown, truncated at 50 KB.

**Flows depending on it:** Flow 6 (Jira Import) exclusively. No other flows call Jira.

**Auth:** Basic Auth — `ATLASSIAN_EMAIL` + `ATLASSIAN_API_TOKEN`. Missing credentials → `status='failed'` on import job.

**Key env vars:** `ATLASSIAN_URL`, `ATLASSIAN_EMAIL`, `ATLASSIAN_API_TOKEN`

---

### Vercel (Hosting + Background Execution)

```
  Browser / API Client         Vercel Platform
          |                           |
          |-- HTTPS request ---------->  Edge Middleware (auth guard)
          |                           |-- Next.js serverless fn
          |                           |
          |                      after() (non-blocking)
          |                           |-- runImportJob(jobId)
          |                           |   (background, after HTTP response)
          |                           |-- Jira pages, DB writes
```

**What it does:** Hosts the Next.js application (SSR + API routes as serverless functions). Edge Middleware runs the auth guard on every request. `after()` runs background work after the HTTP response is sent.

**How it affects data:** No direct data impact. `after()` is the execution vehicle for `runImportJob`; its timeout determines the max Jira import size (exact Vercel timeout for this project unknown — see Discovery Gaps).

**Flows depending on it:** All flows (hosting); Flow 6 (Jira Import, `after()` worker specifically).

---

### Scalar API Reference (Auto-generated Docs)

```
  Developer / AI Agent         GET /api/docs
          |                           |
          |-- browser GET ------------>  Scalar React component
          |                           |-- loads /api/openapi (JSON)
          |<-- interactive docs -------  renders schemas, try-it-out
          |                           |
          |-- GET /api/openapi -------->  force-static route
          |<-- OpenAPI 3.1 JSON --------  public/openapi.json (5-min cache)
```

**What it does:** Auto-generated interactive API documentation using Scalar's React component, backed by an OpenAPI 3.1 spec generated from Zod schemas via `@asteasolutions/zod-to-openapi`.

**How it affects data:** Read-only. The spec is generated by `bun run openapi:gen` and committed as `public/openapi.json`. The live route serves the static file.

**Flows depending on it:** No production flow depends on it. Primarily used by AI agents (QA automation tooling) and developers for API discovery.

---

### Email Service (Unconfirmed)

**What it does:** Delivers magic-link emails and potentially workspace invite emails.

**Evidence:** `RESEND_API_KEY` present in `.env.example`. No Resend call found in application code. Supabase Auth handles OTP email delivery natively — the Resend key may be configured in the Supabase dashboard as a custom SMTP provider rather than called directly from app code.

**How it affects data:** No direct DB data impact. Email delivery failure prevents user onboarding (magic-link) or workspace growth (invites).

> **Discovery Gap**: The actual email provider (Supabase default SMTP vs Resend vs another provider) is not verifiable from the codebase alone. See section below.

---

## Discovery Gaps

| Gap | Where Looked | Impact on Testing |
|-----|-------------|------------------|
| **Plan tier enforcement** | All `app/` routes, `lib/` — no billing/gate code found | Cannot test `cloud`/`enterprise` feature gates; schema has `plan` column but no app logic gates features per plan |
| **`lib/types/supabase.ts` not committed** | Repo root, `lib/types/` — file absent | Full TypeScript types for all tables unavailable until `bun run types:gen` is run; affects type safety of tests |
| **Email delivery provider** | `package.json`, `lib/`, `app/api/` — `RESEND_API_KEY` in `.env.example` but no Resend import found in app code | Cannot confirm transactional email delivery path; magic-link testing requires Supabase Auth email config to be verified separately |
| **Rate limiting** | `middleware.ts`, `lib/api/handler.ts` — `rate_limited` error code defined in `lib/api/error-envelope.ts` but no rate-limit middleware found | No DoS protection confirmed; load testing boundaries unknown |
| **`idempotency_keys` expiry cleanup** | Migrations 0001–0034, `package.json` — no `pg_cron` extension, no cleanup cron | Expired rows (24h TTL) may accumulate indefinitely without a cleanup job |
| **Jira webhook inbound** | `app/api/v1/` route tree — no `/webhooks/` handler | Story status changes in Jira do not automatically sync to Bunkai; import is the only sync mechanism |
| **Background worker timeout** | `lib/jira/import-runner.ts`, Vercel config — Vercel `after()` timeout not confirmed | Large JQL queries (many issues) may silently time out; `MAX_PAGES=1000 × PAGE_SIZE=100 = 100,000 issues` theoretical max |
| **Run verdict transitions (BK-35/39)** | `0031_runs.sql` — `passed`/`failed` transitions noted as "BK-39 verdict targets" (not yet shipped) | Run verdict update API route is not confirmed in codebase snapshot |
| **Monitoring stack** | Repo-wide scan — no Sentry, Datadog, or similar found | Production error observability unknown |
| **Seed mechanism for test data** | `package.json`, `scripts/` — no seed script | Test environment setup requires manual workspace + user creation; first user bootstrapped via auth signup trigger |
| **Activity log event catalog** | `supabase/migrations/0009_cross_cutting.sql` — `activity_log` table exists; `.context/business/events.md` referenced but not found | ATC/Test/Run event types not fully enumerated; activity_log can only be verified against live DB |
| **Connection pool configuration** | `lib/supabase/` — only SDK defaults | Max concurrent DB connections unknown; performance under load untested |
| **Single shared Supabase project across all environments** | `.agents/project.yaml` — `db_project_ref` repeated for local/staging/production | Test data bleed across environments is a HIGH isolation risk for E2E tests |
