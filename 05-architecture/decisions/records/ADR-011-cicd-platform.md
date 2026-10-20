# ADR-011 — CI/CD Platform

- **ID:** ADR-011
- **Date:** 2026-09-12
- **Status:** Superseded by ADR-018 (GitHub Actions — Jenkins was never provisioned; CI runs on `.github/workflows/ci.yml`)
- **Authors:** FaceAttend EDU Team
- **Implementation note (2026-10-20, verified against `ProyectoFaceAttendEDU/.github/`):** **not implemented as written.** There is no Jenkins and no Jenkinsfile. CI and deployment run on GitHub Actions (`ci.yml`, `deploy-production.yml`, `deploy-backend.yml`, `deploy-database.yml`, `deploy-frontend.yml`), with EC2, Neon and Vercel as targets. The decision must be superseded or amended; see `15-project-control/open-questions.md` Q-012.

---

## Context

The system requires a CI/CD platform to automate building, testing, and deployment across four environments: local, dev, staging, and production (per `10-devops/environments.md`). The pipeline must:

1. Run lint, unit tests, and build on every PR.
2. Run integration and contract tests on merge to `develop`.
3. Deploy to dev automatically, staging on merge to `main`, and production with manual approval.
4. Support both TypeScript (Node.js) and Python (biometric-service) build pipelines.
5. Execute Liquibase migrations (ADR-010) before application deployment.
6. Run k6 performance tests in staging.

**Known constraints:**
- The repository is already hosted on GitHub (`.github/` directory with PR template).
- Team of 3 with no dedicated DevOps role — platform must be low-maintenance.
- Self-hosted or cloud-hosted must be decided.

---

## Decision

**We decided:** Use **Jenkins** as the CI/CD platform, deployed as a Docker container.

**Justification:**

1. **Self-hosted control:** Jenkins runs on the team's own infrastructure, with no dependency on external CI/CD services. Data and pipelines stay under the team's control.
2. **Pipeline-as-code:** Jenkinsfile (declarative pipeline) is version-controlled alongside the application code.
3. **Extensible:** Jenkins has plugins for every tool in the stack: Docker, Liquibase, npm, pip, k6, Kong, SSH deployment.
4. **Multi-branch pipeline:** Jenkins supports branch-specific pipelines (PR pipeline, develop pipeline, main pipeline) matching the workflow in `environments.md`.
5. **Free and open-source:** Jenkins is fully free with a massive plugin ecosystem.

**Pipeline structure (per `environments.md`):**

```groovy
// Jenkinsfile (simplified)
pipeline {
    agent any
    stages {
        stage('PR Pipeline') {
            when { changeRequest() }
            steps {
                sh 'npm run lint'
                sh 'npm run test:unit'
                sh 'npm run build'
            }
        }
        stage('Develop Pipeline') {
            when { branch 'develop' }
            steps {
                sh 'npm run test:unit'
                sh 'npm run test:integration'
                sh 'docker build -t faceattend/app:${BUILD_NUMBER} .'
                sh 'docker push faceattend/app:${BUILD_NUMBER}'
                sh 'kubectl apply -f k8s/dev/'
                sh 'npm run test:smoke -- --env=dev'
            }
        }
        stage('Staging Pipeline') {
            when { branch 'main' }
            steps {
                sh 'npm run test:all'
                sh 'npm run test:contract'
                sh 'docker build -t faceattend/app:${BUILD_NUMBER} .'
                sh 'docker push faceattend/app:${BUILD_NUMBER}'
                sh 'liquibase update'
                sh 'kubectl apply -f k8s/staging/'
                sh 'npm run test:smoke -- --env=staging'
                sh 'k6 run tests/performance/load-test.js'
            }
        }
        stage('Production Deploy') {
            when { branch 'main' }
            input { message 'Deploy to production?' }
            steps {
                sh 'kubectl apply -f k8s/production/'
                sh 'sleep 900' // Monitor for 15 min
                sh 'npm run test:smoke -- --env=production'
            }
        }
    }
}
```

---

## Evaluated alternatives

| Alternative | Pros | Cons | Reason for discarding |
|------------|------|------|-----------------------|
| **Jenkins (chosen)** | Self-hosted; pipeline-as-code; massive plugin ecosystem; free; runs on Docker | Requires maintenance (updates, plugins); UI is dated; can consume resources on small servers | — (chosen) |
| GitHub Actions | Already on GitHub; YAML pipelines; generous free tier; zero maintenance | Cloud-hosted (data leaves self-hosted infra); limited self-hosted runner support; less flexible than Jenkins for complex pipelines | Data sovereignty concern; team prefers self-hosted control |
| GitLab CI | Excellent pipeline features; built-in container registry; auto-devops | Requires migrating to GitLab; overkill if already on GitHub | Repository is on GitHub; migration is unnecessary overhead |
| Drone CI | Lightweight; Docker-native; simple YAML config | Smaller plugin ecosystem; less mature than Jenkins; limited multi-branch support | Less ecosystem support; Jenkins is more established |

---

## Consequences

**Positive:**
- Full control over CI/CD infrastructure; no external service dependency.
- Jenkinsfile version-controlled alongside code; changes are reviewable.
- Plugin ecosystem covers every tool in the stack.
- Docker-based deployment simplifies Jenkins setup and maintenance.

**Negative / Trade-offs:**
- Jenkins requires periodic maintenance (plugin updates, security patches).
- Jenkins UI is dated compared to GitHub Actions or GitLab CI.
- Jenkins container consumes resources on the build server.

**Impact on the system:**
- Affected modules: all (CI/CD pipeline applies to all modules).
- Documents that must be updated:
  - `10-devops/environments.md` — reference Jenkins as the CI/CD platform; update pipeline steps
  - `10-devops/local-setup.md` — document Jenkins local access for pipeline debugging
  - `.github/` — Jenkinsfile replaces or supplements GitHub Actions workflows

---

## Risks

| Risk | Probability | Impact | Mitigation |
|------|------------|--------|-----------|
| Jenkins maintenance burden on 3-person team | Medium | Medium | Use Docker official Jenkins image; keep plugins minimal; document maintenance in `14-training/technical-onboarding.md` |
| Jenkins container failure blocks all CI/CD | Low | High | Persistent volume for Jenkins home; Docker restart policy; document recovery in runbook |
| Plugin compatibility issues after updates | Medium | Low | Pin plugin versions; test updates in dev environment first |

---

## References

- Environments and pipeline → `10-devops/environments.md`
- Local setup → `10-devops/local-setup.md`
- Git conventions → `00-governance/git-conventions.md`
- Deployment checklist → `10-devops/environments.md` §Production deploy checklist
