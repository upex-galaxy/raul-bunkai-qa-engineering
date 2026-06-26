# User Story — Format Reference

> **Format-reference only.** This is the canonical shape of a BK User Story. It is NOT a per-ticket authoring target — per-ticket content is synced from Jira (source of truth) by `/sprint-testing` via `bun run jira:sync-issues get <KEY> --include-comments`. Fields marked `[SYNC]` are owned by Jira; never hand-write them locally.

---

## Story Header

| Field | Value |
|---|---|
| **Key** | `[BK-NNN]` |
| **Title** | `[story title from Jira summary]` |
| **Epic Link** | `[EPIC-BK-NNN — epic name]` *(customfield_10014 — Epic Link)* |
| **Sprint** | `[Sprint name]` *(customfield_10020 — Sprint)* |
| **Story Points** | `[N]` *(customfield_10035 — Story Points)* |
| **Assignee** | `[assignee display name]` |
| **Status** | `[current Jira status — e.g. Ready For QA]` |
| **Workflow** | UPEX Feature (US) Workflow |

---

## User Story Statement

> *(customfield_10063 — "✅ Acceptance Criteria (Gherkin)" or story description)* `[SYNC]`

**As a** [persona]
**I want to** [action]
**so that** [benefit]

---

## Acceptance Criteria

> Gherkin format. One scenario per AC. Numbered AC1, AC2, ... `[SYNC]`

### AC1 — [criterion label]

```gherkin
Given [precondition]
When [action]
Then [expected outcome]
```

### AC2 — [criterion label]

```gherkin
Given [precondition]
When [action]
Then [expected outcome]
```

*(Add more ACs as needed)*

---

## AC Quality Checklist

Before this story enters QA, each AC must satisfy:

- [ ] **Specific and measurable** — outcome is observable and unambiguous
- [ ] **Testable** — can be automated or manually verified with pass/fail result
- [ ] **Independent** — does not assume the outcome of another AC
- [ ] **Business-focused** — describes user/business value, not implementation detail

> Shift-Left QA reviews this checklist during the `Shift-Left QA` workflow state. Gaps are raised as comments on the Jira story before estimation.

---

## Technical Notes

> *(customfield_10095 — "🛠️ Spec Implementation Plan (Dev)" or story description)* `[SYNC]`

- [ ] **API changes** — `[endpoint(s) affected, method, contract delta]`
- [ ] **DB changes** — `[table/column/migration, Supabase impact]`
- [ ] **UI changes** — `[component(s), route(s), state affected]`
- [ ] **Dependencies** — `[external service, feature flag, or upstream ticket]`
- [ ] **QA Framework** — *(customfield_10112 — QA Framework)* `[e.g. playwright_javascript]`

---

## Scope

> *(customfield_10119 — "Scope ⛳")* `[SYNC]`

What this story IS expected to deliver:

- [in-scope item 1]
- [in-scope item 2]

---

## Out of Scope

> *(customfield_10075 — "Out Of Scope 🏴")* `[SYNC]`

What this story is NOT going to deliver:

- [exclusion 1]
- [exclusion 2]

---

## Workflow / User Journey

> *(customfield_10082 — "Workflow 🧬")* `[SYNC]`

[User journey narrative — describe the sequence of steps the user takes end-to-end]

---

## Links

### Design / Mockups

> *(customfield_10137 — "Mockup 🎴" / customfield_10081 — "Weblink (URL)")* `[SYNC]`

- Mockup: `[Figma / design tool URL]`
- Weblink: `[external reference URL]`

### Related Issues

Link type names are resolved from `.agents/jira-link-types.json`:

| Link Type | Direction | Jira Name | Use |
|---|---|---|---|
| `test` | outward: `tests` | Test | Coverage: Story is tested by a Test Plan / Test Execution |
| `problem_incident` | outward: `causes` | Problem/Incident | Story caused a Bug/Defect during QA |
| `blocks` | outward: `blocks` | Blocks | Open Bug/Defect blocks this story |
| `relates` | symmetric: `relates to` | Relates | General informational link |
| `test_automation` | outward: `automation test for` | Test Automation | Links manual Test to its automation |

Example:

- `[BK-NNN]` — blocks — `[BK-MMM]` (open defect blocking QA)
- `[BK-NNN]` — tests — `[BK-PPP]` (Test Plan covering this story)

---

## Business Rules

> *(customfield_10116 — "🚩Business Rules Specification")* `[SYNC]`

[Business rules tied to this story, separate from acceptance criteria — domain constraints, compliance requirements, pricing rules, etc.]

---

## Discovery Gaps

> Fill this section during Shift-Left QA if any of the following are missing or ambiguous. Never invent ACs — raise gaps as Jira comments.

- [ ] ACs are missing or written as "TBD" — cannot test without them
- [ ] No mockup link — UI behaviour is ambiguous
- [ ] API contract not specified — cannot write API-layer tests
- [ ] Story Points missing — estimation not complete
- [ ] Out-of-scope not defined — unclear test boundaries
- [ ] `[other gap found]`
