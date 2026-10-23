# ADR-020 — Deployment Platforms: AWS EC2 (backend) + Vercel (web)

- **ID:** ADR-020
- **Date:** 2026-10-17
- **Status:** Accepted
- **Authors:** FaceAttend EDU Team

> **Implementation note (2026-10-20):** the "manual deploy" statements below
> are now half-superseded: `.github/workflows/deploy-backend.yml` performs the
> same SSH + `docker-compose.prod.yml` flow from GitHub Actions (running
> `.github/scripts/deploy-backend-remote.sh` on the host, with health gate and
> Kong check), and `deploy-production.yml` orchestrates database → backend →
> frontend behind the `production` environment approval. The EC2/Neon/Vercel
> platform choice itself is unchanged. The "HTTP only / TLS pending" trade-off
> is addressed in code by ADR-022: `docker-compose.prod.yml` runs a Caddy
> container (`Caddyfile.prod`, Let's Encrypt) in front of Kong, and the EAS
> `preview` profile already targets an `https://` API. Certificate issuance on
> the EC2 host could not be verified from the repository. See `10-devops/ci-cd.md`.

---

## Context

`10-devops/environments.md` defined four environments (local, dev, staging, production) but named no cloud provider and stated that only local existed operationally. That changed on 2026-10-16: the backend was deployed for the first time (recorded in `FULL/DESPLIEGUE.md`), and the web frontend is set up to publish on Vercel. The deployment decision was made in practice before being recorded here, so this ADR captures it.

**Known constraints:**
- The stack is Docker Compose-based: 9 services + Kong + MongoDB/Redis/Kafka; three runtimes (JVM, Node, Python; Go was removed by ADR-021) including JVMs that need real memory.
- The relational model is schema-per-context on a single PostgreSQL instance (ADR-002) — a managed Postgres is a drop-in replacement for the compose one.
- Biometric capture in the browser (kiosk) requires `getUserMedia`, which browsers only grant on HTTPS or `localhost`.
- Team of 3 with no Kubernetes experience; NFR-009 accepts "Docker Compose in an academic staging / Cloud VM" as a valid deployment target.
- The web client needs the Kong URL at build time (`EXPO_PUBLIC_API_URL`).

---

## Decision

**We decided:** Deploy the **backend as Docker Compose on a single AWS EC2 instance**, with **Neon (serverless PostgreSQL, AWS region `sa-east-1`)** replacing the compose Postgres, and publish the **web frontend on Vercel** pointing at the EC2 Kong endpoint. Mobile builds stay on EAS (ADR-019) with the same API URL.

**Justification:**

1. **Compose is the product:** the whole stack (13 containers: 9 services + Kong + Mongo/Redis/Kafka) already starts with one command locally; EC2 runs the identical `docker-compose.yml` files, so local and production stay the same artifact (NFR-009).
2. **Neon removes the Postgres operational burden:** managed backups, pooling (`-pooler` URL) and scale-to-zero for a database that serves a single institution; the 8 Liquibase jobs (ADR-010) run unchanged against it.
3. **EC2 fits the memory profile:** four Java services need ~2 GB each; a `t3.xlarge` (16 GB) runs all 13 containers with room for Kafka and MongoDB.
4. **Vercel is zero-ops for a static SPA:** the web client is an Expo static export (`dist/`); Vercel serves it with CDN, HTTPS and preview deployments per branch — no web server to operate.

**What is deployed (2026-10-16):**
- EC2 `t3.xlarge` (us-east-1, 150 GB EBS gp3): 13 containers via Docker Compose; Kong proxy on `:8000`, admin `:8001` (loopback only in local, open here — pending hardening); Portainer CE on `:9443` for container management.
- Neon PostgreSQL (`faceattend_db`, 8 schemas migrated by Liquibase) — replaces the compose `postgres` service.
- MongoDB, Redis, Kafka remain on the EC2 instance (compose services).
- Vercel: web project built from `front-end/Web` with `EXPO_PUBLIC_API_URL` = `http://<ec2-host>:8000`.

---

## Evaluated alternatives

| Alternative | Pros | Cons | Reason for discarding |
|------------|------|------|-----------------------|
| **AWS EC2 + Neon + Vercel (chosen)** | Reuses compose unchanged; managed Postgres; zero-ops static hosting; cheap for one institution | Single VM = single point of failure; HTTP-only until TLS is configured; manual deploys | — (chosen) |
| AWS ECS/Fargate | Managed scheduling; no VM to patch | Rewrites compose into task definitions; 3 runtimes × memory tuning; team has no ECS experience | Would discard the compose model the whole project is built on |
| Kubernetes (EKS) | Standard for microservices | Operational overhead far beyond a 3-person team; NFR-009 does not require it | Overkill for the current scale |
| All-AWS (S3 + CloudFront for web) | One provider | More moving parts than Vercel for a static SPA; no preview deployments | Vercel is simpler for a static export |
| On-prem / local only | Full control | No reachable environment for the institution; the 2026-10-16 deployment already exists | Superseded by reality |

---

## Consequences

**Positive:**
- A real production environment exists: same compose files as local, managed database, static web on CDN.
- Deploying backend changes = SSH + `docker compose up -d --build`; no pipeline needed yet (ADR-018 keeps CD manual).
- Neon gives backups and connection pooling without operating them.

**Negative / Trade-offs:**
- Single EC2 instance: no redundancy; a VM failure takes the whole platform down (NFR-010 RTO/RPO not yet met).
- HTTP only: `getUserMedia` (biometric kiosk) will not work from the Vercel HTTPS origin against an HTTP API without mixed-content handling — TLS on EC2 (or an HTTPS load balancer) is pending.
- Kong admin `:8001` and data ports were opened in security groups during setup; they must be closed to the public internet.
- Secrets live in `~/.env.prod` on the EC2 box — no secrets manager yet (`environments.md` names Vault/KMS as the target).
- Deploy is manual and not recorded in CI; `DESPLIEGUE.md` in `FULL/` is the runbook.

**Impact on the system:**
- Affected services: all (deployment topology).
- Documents that must be updated: `10-devops/environments.md` (production exists; provider = AWS + Vercel), `10-devops/README.md` (cloud section), `13-operations/` (Portainer as the ops console).

---

## Risks

| Risk | Probability | Impact | Mitigation |
|------|------------|--------|-----------|
| Single-EC2 outage | Medium | High | Snapshots; compose re-deploy is documented; consider a second instance or ECS later |
| Publicly open admin/data ports | High (today) | High | Close security groups to SSH/8000 only; move Mongo/Redis/Kafka behind `BIND_IP` |
| Mixed content blocks biometric capture on web | High (today) | Medium | Terminate TLS on EC2 (Caddy/nginx) or put Kong behind an HTTPS LB; already noted in `DESPLIEGUE.md` next steps |
| Neon latency from us-east-1 app to sa-east-1 DB | Medium | Medium | Keep pooling; measure P95 (NFR-001); consider same-region if it hurts |
| Manual deploys drift from CI | Medium | Medium | Add a deploy job to GitHub Actions (ADR-018) when staging exists |

---

## References

- Deployment record: `FULL/DESPLIEGUE.md` (2026-10-16)
- Environments model: `10-devops/environments.md`
- Related: ADR-018 (CI; CD stays manual), ADR-019 (web on Vercel), ADR-010 (Liquibase against Neon), ADR-007 (Kong as the single edge)
