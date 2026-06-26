# ACCEPTANCE TEST RESULTS (ATR): [ATR] BK-18 — ATC create/edit REST API

**Jira Key:** [BK-95](https://jira.upexgalaxy.com/browse/BK-95)
**Status:** ACTIVE
**Components:** None

> Run results / coverage are NOT synced — read those via xray-cli. This file mirrors the issue description.

---

## Description

# ATR — BK-18: ATC create/edit REST API (re-run 2026-06-20)

> ***SUCCESS:**** ****Verdict******:****** PASSED (GO).**** 12 / 12 test cases passed. The previously blocking defect ****BK-96 is verified fixed end-to-end***. No open defects. BK-18 is recommended for QA sign-off.

## Summary

| Field | Value |
| --- | --- |
| Story | BK-18 — ATC create/edit REST API (POST/PATCH /atcs) |
| Test Plan (ATP) | BK-94 |
| Test Execution (ATR) | BK-95 (this issue) |
| Modality | jira-xray |
| Environment | Staging — `https://staging-upexbunkai.vercel.app/api/v1` |
| Auth | Bearer PAT scope `atc:write` (openapi-testing) |
| Surfaces | API + DB (no UI — UI is BK-19) |
| Date | 2026-06-20 |
| Result | ***12 / 12 PASSED*** |

## Why this re-run

BK-18 was previously tested (2026-06-08) and blocked by ***BK-96**** (happy PATCH returned 412 instead of 200). BK-96 was fixed (PR #30, commit `421a917`) and Closed, but its retest was ****code-review only*** — no E2E API verification was possible at the time (the QA user had no ATCs). This re-run re-verifies the whole story with refactored, parametrized coverage and closes the E2E verification gap on the fix.

## Results matrix

| TC | Xray | Scenario | Result |
| --- | --- | --- | --- |
| TC01 | BK-149 | POST happy create → 201 (layer UI/API/Unit) | PASSED |
| TC02 | BK-150 | Auth/scope gate (401 / 401 / 403) | PASSED |
| TC03 | BK-151 | AC outside US → 422 `ac*outside*user_story` + rollback | PASSED |
| TC04 | BK-152 | Module outside subtree → 422 `module*outside*project_subtree` + rollback | PASSED |
| TC05 | BK-153 | Step positions invalid (parametrized) → 422 `steps*position*invalid` | PASSED |
| TC06 | BK-154 | Request boundaries BVA (title / steps / tags / layer) | PASSED |
| TC07 | BK-155 | Transactional rollback (DB-count invariant) | PASSED |
| TC08 | BK-156 | ***PATCH happy X-If-Match → 200 (BK-96 regression)*** | PASSED |
| TC09 | BK-157 | Optimistic lock X-If-Match (match / stale / absent) | PASSED |
| TC10 | BK-158 | PATCH non-existent → 404 `not_found` | PASSED |
| TC11 | BK-159 | PATCH empty-body no-op → 200, no version bump | PASSED |
| TC12 | BK-160 | Immutable fields (slug / US / module) preserved | PASSED |

## BK-96 regression — fix verified

The fix moved the optimistic-lock version token from the standard `If-Match` header to a custom `X-If-Match` header, because Vercel's edge intercepts RFC-7232 `If-Match` and returns a `412 PRECONDITION_FAILED` platform page before the function result is returned (the mutation still commits at origin). Verified on staging:

| Request | Result |
| --- | --- |
| `PATCH X-If-Match: <current>` (matching) | ***200*** `application/json`, `x-request-id`, version +1, steps cascade-replaced, `atc.updated` logged |
| `PATCH X-If-Match: <stale>` | 409 `conflict` + `details.current_version` |
| `PATCH` (no `X-If-Match`) | 200, lock skipped |
| `PATCH If-Match: <current>` (legacy) | 412 `x-vercel-error` (documented edge limitation; not the contract) |

## DB integrity (DBHub, staging)

- True-zero baseline (0/0/0) before run; each created ATC produced exactly one `atc.created` event (1:1, scoped).
- Transactional rollback verified: a 422 cross-entity POST left counts unchanged (8/16/8) — zero rows across `atcs` / `atc*steps` / `atc*assertions`.
- Cleanup: all test ATCs deleted post-run → 0/0/0, no orphan children. `activity_log` history retained (audit).

## Observations (non-blocking)

- `affected*test*ids` returns `null` where the MVP contract documented `[]`. Minor — recommend dev confirm the intended empty representation.
- Legacy `If-Match` header remains edge-intercepted (412) on Vercel; the working optimistic-lock contract is `X-If-Match`. Documented in ATP BK-94.

## Traceability

Story BK-18 ← **is tested by** ← TC01–TC12 (BK-149…BK-160), all in Test Plan BK-94 and executed under Test Execution BK-95 with shared Pre-Condition BK-161. Defect BK-96 (Closed) permanently covered by TC08 + TC09.

---

## Related Issues

- tests: [BK-18](https://jira.upexgalaxy.com/browse/BK-18) - TMS-ATC API | Create and edit ATCs with steps and assertions

---

## Metadata

- **Created:** 8/6/2026
- **Updated:** 21/6/2026
- **Reporter:** Ely
- **Assignee:** Unassigned

---

_Synced from Jira by sync-jira-issues_

---
_Source: Xray Test Execution [BK-95](https://jira.upexgalaxy.com/browse/BK-95) description · ATR · synced by sync-jira-issues_
