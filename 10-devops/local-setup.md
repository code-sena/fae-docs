# Local Project Setup

> This document ensures that any developer can have a functional local environment
> in less than 1 hour. If it takes longer, the document is incomplete.
> **Rule:** If you have to guess something or look elsewhere, add what's missing here.

---

## Prerequisites

### Universal tools (all stacks)

| Tool | Minimum version | Install from | Verify with |
|------|----------------|--------------|-------------|
| Docker | 24.0+ | docker.com | `docker --version` |
| Docker Compose | 2.20+ | Included with Docker Desktop | `docker compose version` |
| Git | 2.40+ | git-scm.com | `git --version` |

### Project language tool

| Tool | Minimum version | Install from | Verify with |
|------|----------------|--------------|-------------|
| Java (JDK) | 21 (LTS) | adoptium.net | `java --version` |
| Maven | 3.9+ | maven.apache.org | `mvn --version` |
| Node.js | 20+ | nodejs.org | `node --version` |
| npm | 10+ | bundled with Node | `npm --version` |
| Python | 3.12 | python.org | `python --version` |
| Go | 1.22 | go.dev | `go version` |

> Inside Docker none of these is required: compose builds every service image.
> Install them only to run a service from your IDE.

**Stack tool reference:**
- Node.js + TypeScript → [`_stacks/node-typescript.md`](../_stacks/node-typescript.md)
- Java + Spring Boot → [`_stacks/java-spring.md`](../_stacks/java-spring.md)
- Python + FastAPI → [`_stacks/python-fastapi.md`](../_stacks/python-fastapi.md)
- Go → [`_stacks/go.md`](../_stacks/go.md)

**Recommended (not required):**
- Team editor: [VS Code / IntelliJ IDEA / PyCharm / GoLand]
- Postman or Insomnia for manually testing endpoints

---

## Local ports in use

Verify these ports are free before starting:

| Port | Service | Description |
|------|---------|-------------|
| 8080 | kong-gateway (proxy) | Main entry point (Kong OSS 3.6, DB-less) |
| 8081 | ms-identity | Person, user, authentication, sessions (opaque `sessionId`) |
| 8082 | ms-authorization | Roles, permissions |
| 8083 | ms-academic | Schools, programs, courses, actors, enrollments |
| 8084 | ms-scheduling | Environments, schedule blocks, class sessions |
| 8085 | ms-attendance | Attendance records, justifications |
| 8086 | ms-biometric | Facial/fingerprint embeddings |
| 8087 | ms-configuration | Academic/security configs, biometric update cases |
| 8088 | ms-notification | Alerts |
| 8089 | ms-quality | ISO 25010/29110/ISTQB quality tooling |
| 8090 | frontend-web | Expo Web served by nginx |
| 8001 | kong-gateway (admin) | Kong Admin API (loopback, `BIND_IP`) |
| 5432 | PostgreSQL | Relational database (1 instance, 8 schemas) |
| 6379 | Redis | Kong rate-limiting store |
| 27017 | MongoDB | Biometric document store |
| 9092 | Kafka | Domain events (KRaft) |

All ports except **8080** (Kong proxy) and **8090** (frontend) bind to `BIND_IP`
(default `127.0.0.1`). Full reference: `COMPOSE.md` §4.

**Check free ports:**
```bash
# Windows
netstat -ano | findstr ":[PORT]"

# macOS / Linux
lsof -i :[PORT]
```

---

## Step-by-step installation

### Step 1: Clone the repository

```bash
git clone https://github.com/[org]/[repo].git
cd [repo]
```

### Step 2: Environment variables

```bash
# Copy the example environment variables
cp .env.example .env

# Edit .env if you need to change any value
# (for local, the .env.example defaults should work)
```

**Variables in `.env` (copy from `.env.example`; every one has a working default):**

| Variable | Description | Value for local |
|----------|-------------|----------------|
| `POSTGRES_USER` / `POSTGRES_PASSWORD` / `POSTGRES_DB` | PostgreSQL credentials + database | `postgres` / `postgres` / `faceattend_db` |
| `POSTGRES_PORT` | PostgreSQL host port | `5432` |
| `FACEATTEND_LIQUIBASE_IMAGE` | Liquibase image used by the 8 migration jobs | `liquibase/liquibase:4.29.0` |
| `MONGO_USER` / `MONGO_PASSWORD` / `MONGO_DB` / `MONGO_PORT` | MongoDB credentials for biometric vectors | `mongoadmin` / `mongopass` / `faceattend_biometric` / `27017` |
| `REDIS_PORT` / `KAFKA_PORT` | Broker / cache host ports | `6379` / `9092` |
| `KONG_PROXY_PORT` / `KONG_ADMIN_PORT` | Gateway ports | `8080` / `8001` |
| `BIND_IP` | Bind for every non-edge port | `127.0.0.1` (use `0.0.0.0` for LAN/phone testing) |
| `FRONTEND_PORT` | frontend-web host port | `8090` |
| `EXPO_PUBLIC_API_URL` | Backend URL baked into the Web bundle at build time | `http://localhost:8080` |
| `SEED_USERNAME` / `SEED_PASSWORD` | API seed tool credentials | `seed.admin` / `SeedAdmin123!` |
| `BOOTSTRAP_USERNAME` / `BOOTSTRAP_PASSWORD` | Bootstrap admin user | `admin.faceattend` / `Admin123!ChangeMe` |

There is **no** `DATABASE_URL`, `REDIS_URL` or `JWT_SECRET`: compose interpolates each file from
the single root `.env`, and authentication uses opaque session ids issued by ms-identity.

> **NEVER** commit the `.env` file. It is in `.gitignore` for a reason.

### Step 3: Start the stack

```bash
# From the repository root (FULL/): everything, including migrations
docker compose up -d --build

# Verify state + health (Liquibase jobs must be "exited (0)")
docker compose ps
```

Startup order is enforced by the compose file (see `COMPOSE.md` §2):
`postgres` healthy → 8 Liquibase jobs in series
(`identity → authorization → academic → scheduling → attendance → biometric → configuration
→ notification`) → the 9 microservices → `kong-gateway` → `frontend-web`.

Three bridge networks isolate the tiers — `faceattend-edge` (frontend ↔ Kong),
`faceattend-app` (Kong ↔ ms-* ↔ Redis) and `faceattend-data` (ms-* ↔ PostgreSQL/MongoDB/Kafka
+ Liquibase) — so the data tier is unreachable from the browser-facing one (`COMPOSE.md` §3).

Useful subsets:

```bash
docker compose up -d database     # solo datos + migraciones (o: cd database && docker compose up -d)
docker compose up frontend-web kong-gateway   # tras el primer build
docker compose logs -f ms-identity
docker compose down               # para conservando volúmenes
```

### Step 4: Install dependencies

> Commands depend on the project stack. See `_stacks/[your-stack].md → Project commands`.

| Stack | Command |
|-------|---------|
| Node.js + npm | `npm install` |
| Java + Maven | `mvn install -DskipTests` |
| Python + Poetry | `poetry install` |
| Go | `go mod download` |

### Step 5: Run migrations

Migrations are **Liquibase** jobs that run as containers (image
`FACEATTEND_LIQUIBASE_IMAGE`), one per schema, chained in `docker-compose.yml` — you never run
them by hand:

| Job | Schema | Changelog |
|-----|--------|-----------|
| `identity-liquibase` … `notification-liquibase` (8) | `identity, authorization, academic, scheduling, attendance, biometric, configuration, notification` | `database/<NN>-ms-*-db/changelog/changelog-master.yaml` |

```bash
# Applied automatically by `docker compose up -d`; re-run one of them with:
docker compose up identity-liquibase

# Seeds (roles, permissions, actor types, justification types, alert types) are
# changesets inside those same changelogs — see database/SEEDS.md.
# Extra API seed data (optional, Node >= 20):
#   tools/seed-api-test-data  using SEED_USERNAME/SEED_PASSWORD from .env
```

### Step 6: Start the services

> Startup commands depend on the stack. See `_stacks/[your-stack].md → Project commands`.

| Stack | Development command |
|-------|---------------------|
| Node.js | `npm run dev` |
| Java | `mvn spring-boot:run` or `java -jar target/app.jar` |
| Python | `uvicorn main:app --reload` |
| Go | `go run ./cmd/server/...` or `make dev` |

### Step 7: Verify it works

```bash
# Gateway health (the only health route Kong proxies; no GET /health at :8080)
curl http://localhost:8080/health/quality
# → ISO quality report proxied to ms-academic:8083

# Direct probes on the service ports
curl http://localhost:8081/api/v1/health     # ms-identity
curl http://localhost:8083/health            # ms-academic
curl http://localhost:8084/actuator/health   # ms-scheduling (Java)

# Login through the gateway (identifier = email o username)
curl -X POST http://localhost:8080/api/v1/auth/login \
  -H 'Content-Type: application/json' \
  -d '{"identifier":"admin.faceattend","password":"Admin123!ChangeMe"}'
# → {"sessionId":"…","userId":"…","sessionStatus":"Active",…}
```

Then open `http://localhost:8090` (frontend-web). Swagger UI for the Java services:
`http://localhost:8081/swagger-ui.html` (same pattern on 8082/8084/8085).

---

## Daily workflow

```bash
# At the start of the day
git pull origin develop
docker compose up -d            # Liquibase jobs re-run only if there are pending changesets
docker compose ps               # services healthy; migration jobs exited (0)

# Working on one service outside Docker (optional):
#   cd back-end/0X-ms-… && mvn spring-boot:run | npm run dev | uvicorn main:app --reload | go run ./cmd/server

# At the end
git add [specific files]
git commit -m "feat(scope): description"
```

---

## Common issues

### "Port already in use"

```bash
# Find which process is using the port
lsof -i :5432    # macOS/Linux
netstat -ano | findstr :5432  # Windows

# Kill the process (replace PID)
kill -9 [PID]    # macOS/Linux
taskkill /PID [PID] /F  # Windows
```

### "Docker containers not starting"

```bash
# View logs of the failing container
docker compose logs [service-name]

# Restart from scratch (deletes volumes)
docker compose down -v
docker compose up -d
```

### "Database connection refused"

```bash
# Verify PostgreSQL is running
docker compose ps | grep postgres

# View PostgreSQL logs
docker compose logs postgres
```

### "Migration failed"

```bash
# Which Liquibase job failed (they run in series and block the chain)
docker compose ps | findstr liquibase      # Windows
docker compose ps | grep liquibase         # macOS/Linux

# Follow its output (changeset name, SQL error, preconditions)
docker compose logs identity-liquibase

# Re-run it after fixing; the chain resumes on the next `up`
docker compose up identity-liquibase

# Full reference of the 8 changelogs: database/<NN>-ms-*-db/changelog/
```

### "401 Unauthorized on every call"

Authentication is **not** JWT-based: Kong has no `jwt` plugin, so a 401 always comes from the
Java services (`01/02/04/05`) rejecting the session bearer.

```bash
# 1. Get a fresh session id (identifier = email o username)
curl -s -X POST http://localhost:8080/api/v1/auth/login \
  -H 'Content-Type: application/json' \
  -d '{"identifier":"<email-or-username>","password":"<password>"}'

# 2. Send it as a bearer on every /api/** call
curl -s http://localhost:8080/api/v1/persons -H "Authorization: Bearer <sessionId>"

# 3. No roles or a closed session also produce 401/403:
#    GET /api/v1/users/<userId>/roles  ·  GET /api/v1/auth/evaluate?userId=…&permission=…
```

---

## Utility commands

```bash
# Stack (compose)
docker compose up -d --build    # start everything
docker compose ps               # state + health
docker compose logs -f <svc>    # follow one service
docker compose config           # validate the model
docker compose down             # stop keeping volumes

# Tests (per stack, see _stacks/)
mvn test                        # Java services
npm test                        # TS services / frontends
pytest                          # 06-ms-biometric
go test ./...                   # 08-ms-notification

# Database: migrations are the 8 Liquibase jobs (never edit data by hand)
docker compose up identity-liquibase      # re-run one changeset chain
# Reset (DELETES DATA!):  docker compose down -v && docker compose up -d --build

# Code quality
npm run lint && npm run format  # TS/frontends (ESLint + Prettier)
```

---

## Useful local URLs

| Service | URL | Credentials (local only) |
|---------|-----|--------------------------|
| Frontend web (SPA) | http://localhost:8090 | seed users from `tools/seed-api-test-data` |
| API Gateway (Kong proxy) | http://localhost:8080 | `Authorization: Bearer <sessionId>` |
| Kong Admin API | http://localhost:8001 | — (loopback) |
| Swagger UI (Java services) | http://localhost:8081/swagger-ui.html · 8082 · 8084 · 8085 | — |
| PostgreSQL | localhost:5432 (`faceattend_db`) | `.env` `POSTGRES_USER`/`POSTGRES_PASSWORD` |
| MongoDB | localhost:27017 (`faceattend_biometric`) | `.env` `MONGO_*` |
| Redis / Kafka | localhost:6379 / localhost:9092 | — |

There is no RabbitMQ and no Flyway UI: the broker is **Kafka** (domain events) and migrations
are the 8 Liquibase containers.

---

## Correlations

- Environment variables per environment → `10-devops/environments.md`
- CI/CD pipeline → `10-devops/README.md`
- Production troubleshooting → `09-microservices/services/XX/runbook.md`
