# Technical Onboarding - FaceAttendEDU

Welcome to the technical team. This document is your operational guide for your first few days. Our goal is to get you comfortable with the codebase and the workflow so you can make your first contribution within 3 days.

---

## Day 1: Setup and Context

### Morning: Environment & Access
- [ ] **Repository Access:** Confirm your access to the GitHub organization and clones the main repositories.
- [ ] **Communication:** Join the project Slack/Teams channels.
- [ ] **Tooling:** Configure your local environment following `10-devops/local-setup.md`.
- [ ] **Verification:** Run the system and verify that `curl http://localhost:8000/health` responds with `{"status": "ok"}`.

### Afternoon: Core Documentation
Read these documents in the following order:
1. `00-governance/git-conventions.md` — Understanding our branch and commit rules.
2. `01-context/overview.md` — The "Big Picture" of FaceAttendEDU.
3. `02-domain/domain-map.md` — Mapping the business boundaries.
4. `05-architecture/overview.md` — General system design.

---

## Day 2: Domain and Code Exploration

### Business Logic Deep Dive
- [ ] Read `02-domain/entities-and-rules.md` to understand how Attendance and Identity are handled.
- [ ] Review `04-requirements/user-stories.md` to see what the users actually need.
- [ ] Consult `01-context/glossary.md` for specific terms like "Biometric Hash" or "Academic Period".

### Codebase Exploration
- [ ] **Select a Service:** Choose one microservice (e.g., `05-ms-attendance` or `01-ms-identity`).
- [ ] **Analyze Structure:** Verify that the folder structure follows the Hexagonal Architecture:
  - `adapter/in`: Controllers and Request handlers.
  - `application`: Use Cases and Service Interfaces.
  - `domain`: Aggregates, Entities, and Domain Services.
  - `adapter/out`: Repositories and External API implementations.
- [ ] **Test Execution:** Run `mvn test` (Back-end) or `npm run test` (Front-end) to ensure a green baseline.

---

## Day 3: The First Contribution

### The "Good First Issue"
The Tech Lead will assign you a small task (bug fix or doc improvement).
1. **Analyze:** Read the ticket and the acceptance criteria.
2. **Test-First:** Write a failing test for the specific scenario (`🔴 RED`).
3. **Implement:** Write the minimum code to make the test pass (`🟢 GREEN`).
4. **Refactor:** Clean up the code while keeping the test green (`♻️ REFACTOR`).
5. **Submit:** Create a PR following the rules in `00-governance/git-conventions.md`.

### PR Checklist
- [ ] All unit tests pass.
- [ ] Linting is clean.
- [ ] Commit messages follow the `type(scope): description` format.
- [ ] The branch is named correctly (e.g., `feat/add-user-validation`).

---

## 🏗️ Architecture Quick-Reference

To avoid "Architecture Friction", keep these rules in mind:

- **Domain Isolation:** The `domain` package must NOT import anything from `adapter` or `application`.
- **Dependency Rule:** Dependencies only point inwards (Adapter $\rightarrow$ Application $\rightarrow$ Domain).
- **Communication:** Services communicate via the API Gateway or asynchronously through Domain Events.

---

## FAQ

**Q: Where do I ask for help?**
A: Start with the documentation. If you're still stuck after 30 minutes, ask in the team channel.

**Q: Can I change the DB schema directly?**
A: No. Use versioned migrations as defined in `06-data/migration-strategy.md`.

**Q: How do I handle API changes?**
A: Check the OpenAPI contracts in `07-api/` before modifying any endpoint.

---

## Correlations
- **Setup Guide** $\rightarrow$ `10-devops/local-setup.md`
- **TDD Workflow** $\rightarrow$ `11-quality/tdd-guide.md`
- **Git Rules** $\rightarrow$ `00-governance/git-conventions.md`
