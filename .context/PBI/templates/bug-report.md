# Bug Report — Format Reference

> **Format-reference only.** This is the canonical shape of a BK Bug issue. It is NOT a per-ticket authoring target — Bug content is synced from Jira by `bun run jira:sync-issues get <KEY> --include-comments`. Fields marked `[SYNC]` are owned by Jira. `/sprint-testing` authors content in-session and pushes it to the Jira fields listed below; never hand-write `[SYNC]` files locally.

---

## Bug Header

| Field | Jira Field | Value |
|---|---|---|
| **Key** | — | `[BK-NNN]` |
| **Title** | summary | `[one-line bug title]` |
| **Severity** | customfield_10143 — Severity 🚩 | `[critica / mayor / moderada / menor / trivial]` |
| **Priority** | priority | `[Highest / High / Medium / Low / Lowest]` |
| **Error Type** | customfield_10141 — Error Type | `[content / crash / data / functional / integration / performance / security / visual]` |
| **Reporter** | reporter | `[reporter display name]` |
| **Assignee** | assignee | `[assignee display name]` |
| **Sprint** | customfield_10020 — Sprint | `[Sprint name]` |
| **Test Environment** | customfield_10057 — Test Environment 📦️ | `[dev / qa / staging / uat / production]` |
| **Status** | status | `[Open / In Progress / Ready For QA / Closed / …]` |
| **Workflow** | — | UPEX BUG/DEFECT LIFE CYCLE |

---

## One-Line Bug Summary

> *(story description or customfield_10078 — "DESCRIPCIÓN DEL BUG")* `[SYNC]`

**[Component/Feature] — [observed behaviour] when [trigger condition]**

Example: `Login — authentication fails with valid credentials when password contains special characters`

---

## Environment

> *(customfield_10057 — Test Environment 📦️)* `[SYNC]`

| Property | Value |
|---|---|
| Environment | `[dev / qa / staging / uat / production]` |
| URL | `[environment URL — from .agents/project.yaml]` |
| Browser | `[Chrome 125 / Firefox 126 / Safari 17 / Edge 124]` |
| OS | `[macOS 14 / Windows 11 / Ubuntu 22.04]` |
| User Role | `[admin / qa_engineer / regular_user / guest]` |
| Date / Time | `[YYYY-MM-DD HH:MM UTC]` |
| App Version | *(customfield_10086 — VERSIÓN DEL SISTEMA)* `[version string]` |

---

## Steps to Reproduce

> *(customfield_10153 — "Repro Steps (Input)")* `[SYNC]`

1. [Navigate to / open / click ...]
2. [Fill in / select / enter ...]
3. [Submit / confirm / trigger ...]
4. [Observe result]

---

## Expected vs Actual

> *(customfield_10059 — "✅ Expected Result (Output)" / customfield_10056 — "🐞 Actual Result (Comportamiento)")* `[SYNC]`

| | Description |
|---|---|
| **Expected** | `[what should happen per acceptance criteria]` |
| **Actual** | `[what actually happened — exact error message, behaviour, or visual state]` |

---

## Evidence

> *(customfield_10061 — "🧫EVIDENCE")* `[SYNC]`
> Attachments live in: `.context/PBI/bugs/BUG-BK-NNN-<slug>/evidence/` (gitignored)

- **Screenshot**: `[evidence/screenshot-NNN.png]` — `[brief description of what is shown]`
- **Console log**: `[evidence/console-NNN.txt]` — `[error message / stack trace excerpt]`
- **Network request**: `[evidence/network-NNN.har]` — `[failing endpoint, status code, response body]`
- **Video**: `[evidence/repro-NNN.mp4]` — `[repro walkthrough]`
- **Xray Evidence field**: *(customfield_10061)* `[link or inline reference]`

---

## Impact Assessment

| Property | Value |
|---|---|
| **Severity** | `[critica / mayor / moderada / menor / trivial]` — see Severity Guide below |
| **Users Affected** | `[all users / authenticated users / admin only / specific role]` |
| **Frequency** | *(customfield_10122 — FRECUENCY)* `[siempre / ocasional / raro]` |
| **Workaround** | *(customfield_10102 — 🚩 Workaround)* `[steps to unblock users, or "None"]` |
| **In Regression Plan** | *(customfield_10133 — In Regression Plan (QA))* `[yes / no]` |

---

## Severity Guide

| Severity (BK slug) | Jira display | Criteria | Example |
|---|---|---|---|
| `critica` | critica | System unusable, data loss, security breach, payment failure | Cannot log in at all; data deleted on submit |
| `mayor` | mayor | Core feature broken, no workaround available | Cannot create a test plan; test execution crashes |
| `moderada` | moderada | Feature impaired, workaround exists | Filter returns wrong results; manual refresh fixes it |
| `menor` | menor | Minor friction, workaround trivial | Incorrect label on a field; wrong sort order |
| `trivial` | trivial | Cosmetic only, no functional impact | Typo in tooltip; alignment 1px off |

> **Priority vs Severity**: Severity is impact-based (set by QA). Priority is urgency-based (set by PO). They are independent — a cosmetic bug in a security context can be high priority despite low severity.

---

## Fix Classification

> *(customfield_10107 — Fix)* — set after the fix is deployed

| Value | Meaning |
|---|---|
| `bugfix` | Standard fix in the next release |
| `hotfix` | Urgent fix deployed out-of-band |

---

## Root Cause

> *(customfield_10109 — Root Cause 🐞)* — set post-mortem after the fix `[SYNC]`

`[code_error / config_env_error / data_error / environment_error / integration_error / requirement_error / third_party_error / working_as_designed]`

---

## Regression Flag

- [ ] **Regression** — worked in a previous version, broken now
- [ ] **Never worked** — new feature, never functioned correctly
- [ ] **Unknown** — cannot determine from available information

---

## Related Issues

Link type names resolved from `.agents/jira-link-types.json`:

| Link Type | Jira Name | Direction | Use |
|---|---|---|---|
| `problem_incident` | Problem/Incident | `is caused by` (inward) | Story that caused this bug during QA |
| `blocks` | Blocks | `blocks` (outward) | This bug blocks a Story in QA |
| `duplicate` | Duplicate | `duplicates` | This is a duplicate of another bug |
| `test_execute` | Test Execute | `is executed by` | Test Execution that found this bug |
| `relates` | Relates | `relates to` | General informational link |

- `[BK-NNN]` is caused by `[BK-MMM]` (parent story)
- `[BK-NNN]` blocks `[BK-PPP]` (blocked story in QA)

---

## Jira Transition Path

```
Open
  --> In Progress       (start fixing)
  --> In Review         (Pull Request)
  --> Ready For QA      (Fixed & Deployed)
  --> Closed            (ReTest Passed)

  [triage exits]
  --> Cannot Reproduce  (is CNR)
  --> REJECTED          (is WAD)
  --> Duplicated        (is duplicated)
  --> Enhancement       (is not a Bug)
  --> Deferred          (defer)

  [global]
  --> Open              (Re-Open — from any state)
  --> ABORTED           (ABORTED — from any state)
```

---

## Discovery Gaps

- [ ] `VERSIÓN DEL SISTEMA` field (customfield_10086) may not be populated on all bugs — note version manually in Steps to Reproduce when absent
- [ ] `FRECUENCY` options are in Spanish (`siempre`, `ocasional`, `raro`) — ensure QA engineers know the mapping (always / sometimes / rarely)
- [ ] Severity options are in Spanish (`critica`, `mayor`, `moderada`, `menor`, `trivial`) — see Severity Guide above for English mapping
- [ ] `In Regression Plan` flag purpose: confirm with the team whether this drives regression suite inclusion automatically or is advisory only
- [ ] `[other gap found during testing]`
