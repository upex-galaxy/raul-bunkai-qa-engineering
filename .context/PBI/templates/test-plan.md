# Acceptance Test Plan — Format Reference

> **Format-reference only.** This is the canonical shape of the BK Acceptance Test Plan (ATP). It is NOT a per-ticket authoring target — the ATP is authored in-session by `/sprint-testing`, pushed to Jira field `customfield_10067` ("🧪 Acceptance Test Plan (ATP)"), then synced to `.context/PBI/epics/.../stories/STORY-BK-NNN-<slug>/acceptance-test-plan.md`. Never hand-write that synced file. This template is the format reference a QA engineer uses while drafting the ATP content before pushing it to Jira.
>
> **Modality**: BK uses **Modality jira-xray** (`tms_cli: bun xray`). In addition to the Story-level ATP field, a Jira **Test Plan** issue may be created via `/xray-cli` — its description becomes the authoritative ATP source (overrides the Story field on sync).

---

## Header

| Field | Value |
|---|---|
| **Story Key** | `[BK-NNN]` |
| **Story Title** | `[story title]` |
| **Epic** | `[EPIC-BK-NNN — epic name]` |
| **Sprint** | `[Sprint name]` |
| **QA Engineer** | `[name / email]` |
| **Test Date** | `[YYYY-MM-DD]` |
| **Environment** | `[staging / qa / local]` — default: staging |
| **App URL** | `[environment URL from .agents/project.yaml]` |
| **Story Status at Plan** | `[Ready For QA]` |

---

## AC → Test Case Mapping

> Each Acceptance Criterion generates one or more Test Cases (1:N default). Collapse to 1:1 only with a written `trivially atomic` justification. This is the core of the ATP.

| TC ID | AC Ref | Test Case Title | Type | Priority | Automatable |
|---|---|---|---|---|---|
| TC-001 | AC1 | `[test case title]` | Functional | High | Yes |
| TC-002 | AC1 | `[negative / edge case of AC1]` | Functional | Medium | Yes |
| TC-003 | AC2 | `[test case title]` | UI | Medium | Yes |
| TC-004 | AC2 | `[boundary value for AC2]` | Functional | High | No |
| TC-005 | Beyond-AC | `[risk-based case not in ACs]` | Security | High | No |

*(Expand table as needed. AC = floor, not ceiling — add risk-based cases beyond ACs.)*

---

## In Scope

> Derived from `customfield_10119 — Scope ⛳` and AC coverage.

- [feature / flow being tested]
- [API endpoints under test]
- [UI components under test]
- [user roles in scope: admin / qa_engineer / regular_user]

---

## Out of Scope

> Derived from `customfield_10075 — Out Of Scope 🏴` and explicit exclusions.

- [explicitly excluded feature or flow]
- [other environments not tested: production, dev]
- [downstream integrations not tested this sprint]

---

## Test Types

| Type | Required | Reason |
|---|---|---|
| Functional | Yes | Verify ACs and business rules |
| UI / Visual | Conditional | Required when the story includes UI changes |
| API | Conditional | Required when the story touches `app/api/` endpoints |
| Performance | No | Out of scope unless story has explicit perf AC |
| Security | Conditional | Required for auth, permissions, data access |
| Accessibility (A11y) | Conditional | Required for new UI components |

---

## Test Environments

| Environment | URL | When to use |
|---|---|---|
| local | `http://localhost:3000` | Dev smoke only — not a QA gate |
| staging | `https://staging-upexbunkai.vercel.app` | **Default QA environment** |
| production | `https://bunkai.io` | Smoke only, post-release — requires explicit approval |

> Default environment: **staging** (per `testing.default_env` in `.agents/project.yaml`). Credentials from `.env` — never hardcode.

---

## Test Data Requirements

> *(customfield_10074 — TEST DATA)* — document before execution begins

| Data Item | Value / Source | Notes |
|---|---|---|
| User — admin role | `LOCAL_USER_EMAIL` / `STAGING_USER_PASSWORD` from `.env` | Jira: test_environment staging |
| User — regular user | `[env var or fixture]` | |
| Pre-existing record | `[entity type, ID or creation method]` | Required for update/delete tests |
| Edge case value | `[specific input — boundary, special chars, max length]` | |
| API token | From `.env` | Never hardcode |

---

## Test Cases

> Full test case content lives in `.context/PBI/epics/.../stories/STORY-BK-NNN-<slug>/test-cases/TC-NNN.md` (skill-authored, not Jira-synced). This section summarizes the planned TCs.

### TC-001 — [Test Case Title]

| Property | Value |
|---|---|
| **AC Ref** | AC1 |
| **Priority** | High |
| **Type** | Functional |
| **Automatable** | Yes |
| **Precondition** | `[user is logged in as regular_user / record X exists]` |

**Steps:**
1. [step 1]
2. [step 2]
3. [step 3]

**Expected Result:** [expected outcome per AC1]

---

### TC-002 — [Negative Test Title]

| Property | Value |
|---|---|
| **AC Ref** | AC1 — negative |
| **Priority** | Medium |
| **Type** | Functional |
| **Automatable** | Yes |
| **Precondition** | `[precondition]` |

**Steps:**
1. [step 1 — invalid input or forbidden action]
2. [step 2]

**Expected Result:** [error message / blocked action / validation feedback]

---

*(Add TC-003, TC-004, ... following the same structure)*

---

## Edge Cases and Negative Tests

> Beyond the AC-mapping table — risk-based cases that extend coverage past the stated criteria.

| TC ID | Scenario | Technique | AC Ref |
|---|---|---|---|
| TC-E01 | [empty input / null value] | Error Guessing | Beyond AC |
| TC-E02 | [maximum boundary value] | BVA | AC1 |
| TC-E03 | [concurrent action / race condition] | Error Guessing | Beyond AC |
| TC-E04 | [session timeout mid-flow] | State Transition | AC2 |
| TC-E05 | [unauthorized role attempts action] | Decision Table | Beyond AC |

---

## Test Design Techniques Applied

| Technique | Applied | Rationale |
|---|---|---|
| Equivalence Partitioning (EP) | Always | Group valid/invalid input classes |
| Boundary Value Analysis (BVA) | When story has ranges / numeric limits | Test min, max, min-1, max+1 |
| State Transition | When story involves status changes | Model workflow states from `jira-workflows.json` |
| Decision Table | When 2+ conditions interact | Combine AC conditions systematically |
| Pairwise | When 3+ independent variables | Reduce combinatorial explosion |
| Error Guessing | Always | QA domain experience — edge cases |

> Full doctrine: `agentic-qa-core/references/test-design-doctrine.md`

---

## Dependencies, Blockers, Risks

| Item | Type | Status | Owner |
|---|---|---|---|
| `[dependency: upstream story or API contract]` | Dependency | `[open / resolved]` | `[PO / Dev / QA]` |
| `[blocker: environment down / data not seeded]` | Blocker | `[open / resolved]` | `[DevOps / QA]` |
| `[risk: third-party service flaky in staging]` | Risk | `[mitigated / open]` | `[QA]` |

---

## Execution Checklist

### Pre-execution

- [ ] Story status is `Ready For QA` in Jira
- [ ] Build deployed to staging and confirmed by Dev
- [ ] Test data seeded / credentials confirmed from `.env`
- [ ] ATP written and pushed to Jira (`acceptance_test_plan` field or fallback comment)
- [ ] Story transitioned to `In Test` (`Start Testing` transition)

### During execution

- [ ] Each TC executed and result recorded
- [ ] Evidence captured for every failed TC (screenshot, console log, network trace)
- [ ] Bugs filed for each confirmed defect (Jira Bug issue, linked via `Problem/Incident`)
- [ ] Blocked stories: `defect reported` transition fired → story moves to `BLOCKED`

### Post-execution — QA Approved path

- [ ] All TCs passed (or waived with documented justification)
- [ ] ATR written and pushed to Jira (`acceptance_test_results` field or fallback comment)
- [ ] Story transitioned to `QA Approved` (`QA Sign-Off` transition)
- [ ] Test Cases pushed to Xray (via `/xray-cli`) if Modality jira-xray is active

### Post-execution — Blocked path

- [ ] Bug(s) filed and linked to story
- [ ] Story in `BLOCKED` state
- [ ] Workaround documented on bug (if applicable)
- [ ] Dev notified to pick up `Fix defect` transition

---

## Sign-off

| Role | Name | Date | Decision |
|---|---|---|---|
| QA Engineer | `[name]` | `[YYYY-MM-DD]` | `[QA Approved / Blocked]` |
| PO / Reviewer | `[name]` | `[YYYY-MM-DD]` | `[Accepted / Returned]` |

---

## Discovery Gaps

- [ ] No mockup available — UI test cases rely on QA interpretation of ACs
- [ ] API contract not published — API test cases depend on dev confirmation
- [ ] Test data setup process not documented — manual seeding required
- [ ] Performance baseline not defined — performance tests cannot be executed
- [ ] Accessibility (A11y) requirements not stated — A11y tests excluded from scope
- [ ] `[other gap found during planning]`
