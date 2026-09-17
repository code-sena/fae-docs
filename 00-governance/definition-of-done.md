# Definition of Done (DoD)

> A User Story is **DONE** when it meets ALL criteria on this checklist.
> If even one is missing, the story is NOT done — it goes back to In Progress.

## Mandatory checklist

### Code
- [ ] Code implements all acceptance criteria of the user story
- [ ] Code was reviewed and approved by at least 1 team member (PR review)
- [ ] Code follows project standards (linting and formatting pass in CI)
- [ ] No technical debt introduced without registering it in `15-project-control/technical-backlog.md`

### Tests
- [ ] Unit tests written for new business logic (domain invariants + use cases)
- [ ] Test coverage ≥70% and does not decrease from baseline
- [ ] All tests pass locally and in CI
- [ ] Acceptance criteria verified (manual or automated, Given/When/Then)

### Integration
- [ ] Changes do not break other modules (integration tests pass)
- [ ] If API changes: OpenAPI contract updated in `07-api/contracts/openapi/` + `_shared.yaml` if needed
- [ ] If data model changes: service `data-model.md` + `06-data/domains/` updated, migration via Flyway/Liquibase (ADR-010)
- [ ] If new/modified events: `02-domain/domain-events.md` + `09-microservices/event-catalog.md` + per-service `events.md` updated

### Deployment
- [ ] Code is mergeable to `dev` (no conflicts)
- [ ] CI/CD green on the branch
- [ ] Deployed to staging environment
- [ ] Basic smoke test passing on staging

### Documentation
- [ ] Service `README.md` updated if the public interface changed
- [ ] If a significant technical decision was made: ADR created or updated

---

## Allowed exceptions

The following exceptions must be explicitly agreed to by the Tech Lead:
- E2E tests omitted due to environment limitations (document the risk)
- Documentation deferred for urgent delivery (create a tech-debt ticket)

---

## What is NOT a Done criterion

- "The code is on my machine" — it must be in the repository
- "It works on my local environment" — it must work on staging
- "The PM/PO approved it" — that is the product Definition of Done, not the code's
