# PBI Backlog Access Recipe — Bunkai (BK)

> **This is the BK project-specific backlog access recipe.** Per-ticket PBI is NOT stored here — it is synced on demand from Jira by `/sprint-testing` via `bun run jira:sync-issues get <KEY> --include-comments`. The `epics/` subtree is a read-only Jira cache materialized at runtime, not authored here.

| Field | Value |
|---|---|
| PM Tool | Jira Cloud |
| Project Key | BK |
| Workflow Scheme | UPEX PROGRAM Workflow Scheme |
| Board Type | Scrum (sprint-based) |
| Primary Access | `/acli` skill (CLI) |
| Fallback Access | Atlassian MCP |
| Last Updated | 2026-06-24 |

---

## Backlog Location

| Item | Value |
|---|---|
| Jira Instance | `$ATLASSIAN_URL` (resolved from `.env`) |
| Project URL | `$ATLASSIAN_URL/jira/software/projects/BK/boards` |
| Project Key | BK |
| Board Name | Not verified from catalog — check Jira board settings (see Discovery Gaps) |

---

## Access Configuration

### Primary: `/acli` (CLI)

Load the `/acli` skill before any `[ISSUE_TRACKER_TOOL]` call. The CLI issues REST calls authenticated via the env vars below.

```bash
# Example — list open stories ready for QA
acli jira issue list --project BK --status "Ready For QA" --type Story
```

### Fallback: Atlassian MCP

The Atlassian MCP server is configured in `.mcp.json`. Use it when `acli` is unavailable or when the operation benefits from rich MCP integration (e.g., complex JQL with nested fields).

### Required Environment Variables

```
# .env — never paste values here; use this as the key reference
ATLASSIAN_URL=
ATLASSIAN_EMAIL=
ATLASSIAN_API_TOKEN=
```

All three must be set before any `acli` or Atlassian MCP call. Missing any one → MCP auth failure or `acli` 401 → STOP immediately and ask user to fix `.env` + restart the agent session.

---

## Project Structure

### Issue Types

| Issue Type | Jira Name | Coverable | Workflow | Local Dir |
|---|---|---|---|---|
| Story | Story | Yes | UPEX Feature (US) Workflow | `epics/.../stories/` |
| Bug | Bug | Yes | UPEX BUG/DEFECT LIFE CYCLE | `bugs/` |
| Epic | Epic | No (container) | UPEX Epic Workflow | `epics/` |
| Defect | Defect | Yes | UPEX BUG/DEFECT LIFE CYCLE | `defects/` |
| Improvement | Improvement | Yes | UPEX BUG/DEFECT LIFE CYCLE | `improvements/` |
| Tech Story | Tech Story | Yes | UPEX Tech Task/Debt (TD) Workflow | `tech-stories/` |
| Tech Debt | Tech Debt | Yes | UPEX Tech Task/Debt (TD) Workflow | `tech-debts/` |
| Test (TC) | Test | No | UPEX Test (TC) Workflow | `tests/` |
| Test Plan | Test Plan | No (Xray ATP) | UPEX Test Planning Workflow | `test-plans/` |
| Test Execution | Test Execution | No (Xray ATR) | UPEX Subtask/TX Workflow | `test-executions/` |
| Re-Test Execution | Re-Test Execution | No (Xray ATR) | UPEX Subtask/TX Workflow | `test-executions/` |
| Test Set | Test Set | No (Xray grouping) | UPEX Test Suite (TS) Workflow | `test-sets/` |
| Precondition | Precondition | No (Xray prereq) | UPEX Active Item Workflow | `preconditions/` |

> Xray issue types (Test, Test Plan, Test Execution, Re-Test Execution, Test Set, Precondition) confirm **Modality jira-xray** is in use. TMS operations use `/xray-cli`; generic Jira operations use `/acli`.

### Story Workflow — UPEX Feature (US) Workflow

```mermaid
stateDiagram-v2
    [*] --> Backlog : Create

    Backlog --> Shift-Left_QA : Analyze
    Backlog --> Estimation : Ready to Estimate

    Shift-Left_QA --> Estimation : Estimate
    Shift-Left_QA --> Backlog : back

    Estimation --> Ready_For_Dev : Estimated and Ready to work
    Estimation --> Backlog : back
    Estimation --> Shift-Left_QA : needs quality

    Ready_For_Dev --> In_Progress : Start working

    In_Progress --> In_Review : Pull Request
    In_Progress --> Ready_For_QA : Pushed
    In_Progress --> Ready_For_Dev : back

    In_Review --> Ready_For_QA : Deployed

    Ready_For_QA --> In_Test : Start Testing

    In_Test --> QA_Approved : QA Sign-Off
    In_Test --> Ready_For_QA : back
    In_Test --> BLOCKED : defect reported

    BLOCKED --> In_Test : back
    BLOCKED --> In_Progress : Fix defect
    BLOCKED --> Ready_For_Dev : back to dev

    QA_Approved --> Ready_For_Release : include in release
    QA_Approved --> In_Test : back

    Ready_For_Release --> Deployed_to_Production : released

    Deployed_to_Production --> [*]
    QA_Approved --> [*]
    ABORTED --> [*]

    Backlog --> ABORTED : ABORTED
    In_Progress --> ABORTED : ABORTED
    ABORTED --> Ready_For_Dev : Recover
```

**QA-relevant states (real Jira names):**

| Canonical Slug | Real Jira Status Name |
|---|---|
| `backlog` | Backlog |
| `shift_left_qa` | Shift-Left QA |
| `estimation` | Estimation |
| `ready_for_dev` | Ready For Dev |
| `in_progress` | In Progress |
| `in_review` | In Review |
| `ready_for_qa` | Ready For QA |
| `in_test` | In Test |
| `blocked` | BLOCKED |
| `qa_approved` | QA Approved |
| `ready_for_release` | Ready For Release |
| `deployed_to_production` | Deployed to Production |
| `aborted` | ABORTED |

### Bug / Defect Workflow — UPEX BUG/DEFECT LIFE CYCLE

| Canonical Slug | Real Jira Status Name | Category |
|---|---|---|
| `open` | Open | new |
| `in_progress` | In Progress | indeterminate |
| `in_review` | In Review | indeterminate |
| `deferred` | Deferred | indeterminate |
| `ready_for_qa` | Ready For QA | new |
| `closed` | Closed | done |
| `duplicated` | Duplicated | done |
| `rejected` | REJECTED | done |
| `cannot_reproduce` | Cannot Reproduce | done |
| `enhancement` | Enhancement | done |
| `aborted` | ABORTED | done |

### Test Case Workflow — UPEX Test (TC) Workflow

| Canonical Slug | Real Jira Status Name | Category |
|---|---|---|
| `draft` | Draft | new |
| `in_design` | In Design | indeterminate |
| `ready` | READY | done |
| `in_review` | In Review | indeterminate |
| `candidate` | Candidate | new |
| `in_automation` | In Automation | indeterminate |
| `pull_request` | Pull Request | indeterminate |
| `automated` | AUTOMATED | done |
| `manual` | MANUAL | done |
| `deprecated` | DEPRECATED | done |

### Epic Workflow — UPEX Epic Workflow

| Real Status | Category |
|---|---|
| Backlog | new |
| Planning | indeterminate |
| In Progress | indeterminate |
| Done | done |
| ABORTED | done |

---

## Common Queries

> Always load `/acli` skill before running these. Replace `{{PROJECT_KEY}}` with `BK`.

### 1. Current sprint — ready for QA

```
project = BK AND sprint in openSprints() AND status = "Ready For QA" AND issuetype = Story ORDER BY priority DESC
```

`[ISSUE_TRACKER_TOOL]` pseudocode:
```
[ISSUE_TRACKER_TOOL] List Issues:
  project: BK
  sprint: openSprints()
  status: "Ready For QA"
  type: Story
  order_by: priority DESC
```

### 2. All open bugs (unresolved)

```
project = BK AND issuetype = Bug AND status = "Open" ORDER BY priority DESC
```

### 3. Stories currently in test (assigned to me)

```
project = BK AND issuetype = Story AND status = "In Test" AND assignee = currentUser()
```

### 4. Recently updated (last 24 hours)

```
project = BK AND updated >= -1d ORDER BY updated DESC
```

### 5. Stories blocked by open defects

```
project = BK AND issuetype = Story AND status = "BLOCKED" ORDER BY priority DESC
```

### 6. Open Test Plans (Xray ATP queue)

```
project = BK AND issuetype = "Test Plan" AND status in ("Planning", "READY") ORDER BY created DESC
```

### 7. Shift-Left QA queue (pre-sprint grooming)

```
project = BK AND issuetype = Story AND status = "Shift-Left QA" ORDER BY rank ASC
```

---

## Integration with KATA

| Skill trigger | When to fetch | Fetch command |
|---|---|---|
| `/sprint-testing` — start | Sync the target ticket (full content) | `bun run jira:sync-issues get <KEY> --include-comments` |
| `/sprint-testing` — bug triage | Pull bugs linked to the story | `bun run jira:sync-issues jql "issueFunction in linkedIssuesOf('<KEY>', 'is caused by')"` |
| `/shift-left-testing` — batch grooming | Pull a sprint's stories | `bun run jira:sync-issues pull --sprint active` |
| `/test-documentation` — ROI scoring | Pull Test issues for the story | Via `/xray-cli` (Modality jira-xray) |
| `/test-automation` — resume | Read `ROADMAP.md` + `PROGRESS.md` | Already local in `test-specs/` |
| `/regression-testing` — GO/NO-GO | Pull recent Test Executions | `bun run jira:sync-issues pull --types test_execution,re_test_execution` |

### Local storage layout

```
.context/PBI/
|-- README.md                   # this file — backlog access recipe + common queries
|-- templates/                  # format-reference guides (NOT per-ticket targets)
|   |-- user-story.md
|   |-- bug-report.md
|   `-- test-plan.md
`-- epics/                      # synced from Jira by /sprint-testing — READ-ONLY CACHE
    `-- EPIC-BK-<N>-<slug>/
        `-- stories/
            `-- STORY-BK-<N>-<slug>/
                `-- ...         # materialized by `bun run jira:sync-issues get <KEY> --include-comments`
```

> Per-ticket PBI is **never authored here**. It is materialized by `/sprint-testing` from Jira and can always be re-synced. The `templates/` files in this directory are canonical shape references for human use — not per-ticket authoring targets.

---

## Credentials

| Env var | Purpose |
|---|---|
| `ATLASSIAN_URL` | Jira Cloud instance URL (e.g. `https://<workspace>.atlassian.net`) |
| `ATLASSIAN_EMAIL` | Email of the Jira user agent acts as |
| `ATLASSIAN_API_TOKEN` | Jira API token for that user |

All three are set in `.env` (never committed). See `.env.example` for the key names. **Never paste token values in markdown, chat, or code.**

---

## Discovery Gaps

| Gap | Impact | Resolution |
|---|---|---|
| Board name not verifiable from catalog files | `[BACKLOG_BOARD]` cannot be confirmed | Check Jira → Projects → BK → Board settings, or run `acli jira board list --project BK` |
| Sprint cadence (length, current sprint name/ID) not in catalog | JQL using sprint IDs may need adjustment | Run `acli jira sprint list --project BK` to get active sprint; confirm naming convention |
| `formal_blocked_gate: true` in `project.yaml` — assumes `BLOCKED` status is exposed in the Story workflow | If the status is ever removed, the `defect_reported` transition breaks | Verify the story transitions in Jira; confirm `defect reported` (id: 13) is still live |
| Workflow diagram reflects catalog data — not verified against live board | Board may filter some states | QA engineer should transition one test story through the full flow and confirm all states appear |
| No acli board metadata to confirm Scrum vs Kanban cadence | Sprint queries assume Scrum | Run `acli jira board list --project BK` and confirm `type: scrum` |
