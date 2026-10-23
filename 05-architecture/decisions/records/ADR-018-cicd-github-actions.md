# ADR-018 — CI on GitHub Actions

- **ID:** ADR-018
- **Date:** 2026-10-17
- **Status:** Accepted
- **Supersedes:** ADR-011 (CI/CD Platform — Jenkins)
- **Authors:** FaceAttend EDU Team

> **Implementation note (2026-10-20):** deploy automation now exists:
> `.github/workflows/deploy-production.yml` orchestrates CI → database (Neon)
> → backend (EC2) → frontend (Vercel) on push to `main`, each step gated by the
> manual approval of the `production` GitHub environment, with reusable
> `deploy-database/backend/frontend.yml` workflows. The "CD stays manual"
> statements below reflect the 2026-10-17 reality. Workflow files are present on
> `main` and `develop` (verified 2026-10-23).
> Also: Dependabot is **paused** (`open-pull-requests-limit: 0`, monthly,
> no `gomod` entry since the Go service was removed) — it does not open the
> weekly update PRs mentioned in the justification. See `10-devops/ci-cd.md`.

---

## Context

ADR-011 chose Jenkins, self-hosted, as the CI/CD platform. That decision was never implemented: no Jenkins server was ever provisioned, no `Jenkinsfile` exists in the repository, and the team of 3 has no capacity to operate one. Meanwhile `.github/workflows/ci.yml` runs on GitHub Actions for every push/PR to `main`, and `.github/dependabot.yml` covers the eight ecosystems (maven, npm ×6, pip, gomod, docker ×10, github-actions). The deployment story also moved: the backend now runs on AWS EC2 with Docker Compose (see ADR-020) and is deployed manually, so a CD tool with Kubernetes targets is not what the project needs right now.

**Known constraints:**
- The repository is hosted on GitHub; PRs and reviews already happen there.
- Team of 3 with no dedicated DevOps role (carried over from ADR-011).
- Three language runtimes must build in CI: Java 21, Node 20, Python 3.12 (with dlib compiled from source). *(A fourth, Go 1.22, was listed here until 2026-10-18, when `08-ms-notification` moved to TypeScript — ADR-021. No Go job was ever present in `ci.yml`.)*
- CD (deploy automation) is still out of scope: deploys to EC2 are manual `docker compose` over SSH (ADR-020).

---

## Decision

**We decided:** Use **GitHub Actions** as the CI platform. Continuous deployment stays manual (ADR-020) until there is more than one environment to deploy to.

**Justification:**

1. **Already running and green:** `ci.yml` (171 lines) tests all nine services plus both frontends on every push/PR to `main`, with `concurrency: cancel-in-progress`. Switching platforms would discard working pipelines to satisfy a preference recorded on paper.
2. **Zero infrastructure:** no server to provision, patch or keep online — the sovereignty concern that drove ADR-011 does not outweigh the maintenance cost for a team of 3.
3. **Matrix builds per stack:** the workflow uses a matrix per service (`java-services`, `node-services`, `python-services`, `mobile`, `web`), so each service keeps its own toolchain and cache (Maven, npm, Poetry `.venv` keyed by `poetry.lock`).
4. **Dependabot included:** `.github/dependabot.yml` is configured for every ecosystem in the monorepo (currently paused: monthly, limit 0), which a self-hosted Jenkins would not provide out of the box.

**Current pipeline (what `ci.yml` actually does):**
- `java-services` (01, 02, 04, 05): Temurin 21 + `mvn -B test`.
- `node-services` (03, 07, 08, 09): Node 20 + `npm ci` + `npm test` + `npm run build`.
- `python-services` (06): Python 3.12 + Poetry 1.8.3 + `pytest` (build-essential/cmake/libopenblas for dlib).
- `mobile` / `web`: `npm ci` + `npm test` (+ `npx expo export --platform web` for web).
- Security scans (`npm audit`, `bandit`) run as informative steps (`continue-on-error: true`) — gating on them is pending (see `11-quality/testing-strategy.md`).

> **Correction (2026-10-20):** an earlier version of this note listed a `go-service` job
> (`go test ./...`) and a `govulncheck` scan for `08-ms-notification`. **Neither ever existed in
> `ci.yml`** — the workflow has only the five jobs above. After the 2026-10-18 rewrite (ADR-021)
> the notification service is covered by the `node-services` matrix instead. The workflow is
> 157 lines.

---

## Evaluated alternatives

| Alternative | Pros | Cons | Reason for discarding |
|------------|------|------|-----------------------|
| **GitHub Actions (chosen)** | Already running; zero ops; native matrix/cache; Dependabot; PR checks where reviews happen | Hosted on GitHub (sovereignty concern from ADR-011); minutes quota on private repos | — (chosen) |
| Jenkins (ADR-011) | Self-hosted control; huge plugin ecosystem | Requires a dedicated server nobody operates; would replace working pipelines; team of 3 cannot maintain it | Never provisioned in 5+ weeks; GitHub Actions proved sufficient |
| GitLab CI | Good YAML pipelines; self-hostable | The repo is not on GitLab; migrating hosting just for CI adds friction | Repo already lives on GitHub |
| Drone / Gitea Actions | Lightweight, self-hosted | Same server-ops burden as Jenkins for a smaller ecosystem | Same objection as Jenkins |

---

## Consequences

**Positive:**
- CI keeps working with no infrastructure to operate.
- Caches (Maven, npm, Poetry) are hosted and keyed per lockfile, keeping runs fast.
- Dependabot covers security updates for the configured ecosystems (paused, see the implementation note above).

**Negative / Trade-offs:**
- Pipeline logs and secrets live on GitHub — the sovereignty argument of ADR-011 is consciously given up.
- `npm audit` / `bandit` / `govulncheck` are informative, not gating; vulnerability handling depends on someone reading the logs.
- No CD: deploys to EC2 are manual over SSH (ADR-020). If a second environment appears, deploy automation must be designed (GitHub Actions deploy jobs, or a small `deploy.yml` with SSH).

**Impact on the system:**
- Affected services: all (CI covers every service and both frontends).
- Documents updated with this decision: ADR-011 (marked Superseded), `10-devops/environments.md` and `10-devops/README.md` (they now record the GitHub Actions CI and the manual-deploy reality). `10-devops/ci-cd.md` remains the target model for deploy automation.

---

## Risks

| Risk | Probability | Impact | Mitigation |
|------|------------|--------|-----------|
| GitHub outage blocks CI | Low | Medium | Merges can wait; workflows are plain YAML and portable to any runner |
| Informative security scans go unread | Medium | High | Promote `npm audit`/`bandit` to gating steps once baselines are clean |
| Minutes quota on private repos | Low | Low | Build times are small (tests only, no image push); public repo would be free |

---

## References

- Supersedes: ADR-011 → `05-architecture/decisions/records/ADR-011-cicd-platform.md`
- Pipeline: `FULL/.github/workflows/ci.yml`, `FULL/.github/dependabot.yml`
- Deployment (still manual): ADR-020 → `05-architecture/decisions/records/ADR-020-deployment-platforms.md`
- Target pipeline model: `10-devops/environments.md`
