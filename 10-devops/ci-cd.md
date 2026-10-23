# CI/CD

Continuous integration and delivery for FaceAttend-Edu: GitHub Actions workflows,
per-stack pipelines, deployable artifacts, and release gates. All automation
lives in `.github/` of the `FULL` repository (`workflows/`, `scripts/`,
`dependabot.yml`).

## 1. Workflows

| Workflow | File | Trigger | Purpose |
|---|---|---|---|
| CI | `ci.yml` | Push and PR to `main` (also callable) | Test and build every stack |
| Deploy production | `deploy-production.yml` | Push to `main`, or manual dispatch | Orchestrates the full release |
| Deploy database | `deploy-database.yml` | Reusable + manual dispatch | Liquibase migrations on Neon |
| Deploy backend | `deploy-backend.yml` | Reusable + manual dispatch | Microservices + Kong on EC2 |
| Deploy frontend | `deploy-frontend.yml` | Reusable + manual dispatch (+ validation branch push) | Web SPA on Vercel |

The three deploy workflows are **reusable** (`workflow_call`): they are invoked
by `deploy-production.yml` and can also be run alone via `workflow_dispatch`.

## 2. CI pipeline (`ci.yml`)

One job per technology stack, so failures stay isolated (`fail-fast: false`
on every matrix):

| Job | Scope | Steps |
|---|---|---|
| `java-services` | `01-ms-identity`, `02-ms-authorization`, `04-ms-scheduling`, `05-ms-attendance` | `mvn -B test` (JDK 21, Maven cache) |
| `node-services` | `03-ms-academic`, `07-ms-configuration`, `08-ms-notification`, `09-ms-quality` | `npm ci` → `npm test` → `npm run build` |
| `python-services` | `06-ms-biometric` | System build deps (dlib/OpenBLAS), Poetry install, `pytest` (venv cached on the lockfile) |
| `mobile` | `front-end/Mobile` | `npm ci` → `npm test` |
| `web` | `front-end/Web` | `npm ci` → `npm test` → `expo export --platform web` |

Gate characteristics:

- `npm audit --audit-level=high` and `bandit` run as **informational** steps
  (`continue-on-error: true`): they report findings without breaking the
  pipeline (pre-existing transitive debt is tracked by Dependabot instead).
- There is **no lint or formatting gate in CI**; ESLint, TypeScript and Ruff
  are enforced locally only.
- The Kong gateway configuration (`99-api-gateway`) has no dedicated CI job.
- Concurrency: one run per ref, with `cancel-in-progress` (a newer push to the
  same branch cancels the in-flight run).

## 3. Production deploy (`deploy-production.yml`)

A push to `main` starts the full release; each deploy job runs inside the
GitHub **`production` environment, which requires manual approval**. Order is
strict (a failure stops the chain):

```
ci ──> database (Neon) ──> backend (EC2) ──> frontend (Vercel)
```

> **Activation:** all five workflow files are present on both `develop` and `main`
> (verified 2026-10-23 with `git ls-tree origin/main .github/workflows/`).
> `deploy-frontend.yml` still carries a validation push trigger on
> `feature/github-actions-deploy`; it was only needed before the file landed on the
> default branch and is preview-only.

- **Concurrency:** a single production deploy at a time (no cancellation).
- **Manual dispatch** allows deploying a subset: `all`, `database`, `backend`
  or `frontend`.
- **Database** (`deploy-database.yml`): runs the Liquibase containers of
  `database/docker-compose.neon.yml` against **Neon**. Only `update` is ever
  executed — no automatic rollback or drop — and every container's exit code
  is verified before reporting success.
- **Backend** (`deploy-backend.yml`): connects to **EC2 over SSH** (runner's
  own `ssh`/`scp`, no third-party key actions), optionally syncs the
  `.env.prod` file from a secret, then executes
  `.github/scripts/deploy-backend-remote.sh` on the host: checkout of the
  deployed SHA, `docker compose -f docker-compose.prod.yml up -d --build`,
  health wait (up to 10 minutes, excluding one-shot Liquibase containers) and
  a Kong response check on port 8000. **Rollback is manual**: re-run the
  script with the previous SHA.
- **Frontend** (`deploy-frontend.yml`): builds are done remotely by **Vercel**
  (the CLI only triggers the deploy; `EXPO_PUBLIC_API_URL` comes from Vercel's
  environment variables). `preview` and `production` targets exist.

## 4. Dependency automation (`dependabot.yml`)

Dependabot is configured for **Maven, npm, pip (Poetry), Docker and
GitHub Actions**, one entry per directory, with these policies:

- **Currently paused**: `open-pull-requests-limit: 0` on every entry — it
  reports security alerts but opens no update PRs. Reactivating means raising
  the limit.
- Monthly schedule, PRs against `develop` (never `main`).
- Minor and patch updates grouped per directory; **major updates are ignored**
  (runtime upgrades are platform decisions done by hand).
- Java/Python/Node Docker base images are fully ignored for the same reason.

## 5. Secrets and configuration

Workflows receive their configuration from **GitHub environment variables and
secrets**: `EC2_HOST`, `EC2_USER`, `EC2_DEPLOY_DIR`, `EC2_SSH_KEY`,
`EC2_KNOWN_HOSTS`, `ENV_PROD_FILE` (backend); `NEON_JDBC_URL`, `NEON_USER`,
`NEON_PASSWORD` (database); `VERCEL_TOKEN`, `VERCEL_ORG_ID`,
`VERCEL_PROJECT_ID` (frontend). Missing values fail fast with an explicit
error only in the backend and database workflows (their "Validar" steps);
`deploy-frontend.yml` does not validate the Vercel values. Local equivalents are documented in
[Local setup](./local-setup.md); environment semantics in
[Environments](./environments.md).

## 6. Related decisions

- [ADR-018: CI on GitHub Actions](../05-architecture/decisions/records/ADR-018-cicd-github-actions.md)
  records the CI platform decision (and its correction on the never-existing
  Go job — `08-ms-notification` is now covered by the `node-services` matrix).
- [ADR-021: Notification service — TypeScript](../05-architecture/decisions/records/ADR-021-notification-service-typescript.md)
  records the migration decision and its status.
- [ADR-020: Deployment platforms](../05-architecture/decisions/records/ADR-020-deployment-platforms.md)
  covers EC2, Neon and Vercel as the production stack.
