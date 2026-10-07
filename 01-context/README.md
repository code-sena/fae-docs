# 01 — Project Context

> **What is this?** The "why" of the system. Anyone new must be able to read this
> folder and understand what problem the project solves, what it includes, and what it does NOT include.

## Why this section exists

Before designing anything, the team needs to agree on:
- What problem are we solving?
- For whom?
- What is in scope and what is out of scope?
- What does each term we use mean?

Without this, each team member works with different assumptions and the project fragments.

---

## What is here and current fill status (v1.1)

| File | Status | Summary |
|---|--------|---------|
| `overview.md` | ✅ Complete | FaceAttend EDU definition, institutional context SENA, 8 bounded contexts, technology stack (PG/Mongo/Redis/Kong/Kafka-ready), status Under construction MVP |
| `scope.md` | ✅ Complete | In Scope MVP mapped to RF 1-8, Out of Scope (IoT future, Audit retired, payroll/grading), Assumptions/Constraints with Law 1581 |
| `glossary.md` | ✅ Complete | 25+ domain terms + 7 technical terms + acronyms; source of truth for ubiquitous language |

> **No placeholders remain.** All `01-context` docs were completed in `docs/improve-00-01-context-governance` to address evaluation “no muy bien hechas”.

## File purposes (for reference)

- `overview.md` — Executive 1-page: What is FaceAttend EDU, problem, users, technology stack, status.
- `scope.md` — Boundaries: In Scope MVP vs Out of Scope vs Future candidates; Assumptions/Constraints.
- `glossary.md` — Dictionary: domain + technical terms + acronyms; source of truth for `02-domain` ubiquitous language.

---

## Correlations with other sections

| If you change this... | Also review... |
|-----------------------|----------------|
| The problem described in `overview.md` | Product vision in `03-product/vision.md` |
| The scope in `scope.md` | Requirements in `04-requirements/`, PRD in `03-product/` |
| A term in `glossary.md` | Every document where that term appears |

---

## Recommended fill order

1. `overview.md` — 30 minutes with the full team
2. `scope.md` — 1 hour of discussion (the most valuable thing you can do at the start)
3. `glossary.md` — grows throughout the project, start with 10 key terms

---

## Questions this section must answer

- What does this system exist for?
- Who are the users?
- What does the system NOT do?
- What does [term X] mean in this project?
