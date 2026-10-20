# Backup & Recovery — FaceAttend-Edu

> **Current state (verified 2026-10-17):** backups are **manual**, driven by
> `FULL/database/scripts/backup.sh`. There is **no automated retention/rotation** today and no
> off-host scheduling — the script must be run by hand with the stack up. In **production**
> (AWS EC2, ADR-020) the relational layer is **Neon**, which provides managed backups and
> point-in-time recovery for `faceattend_db`; MongoDB and Redis still run on the EC2 host and
> are the operator's responsibility.

---

## What is backed up

| Data store | Contains | Tool | Artifact |
|---|---|---|---|
| PostgreSQL 17 (`faceattend_db`, 8 schemas) | All relational data (identity, authorization, academic, scheduling, attendance, biometric case, configuration, notification) | `pg_dump --format=custom` | `faceattend_db.dump` |
| MongoDB 7 | Biometric embeddings (`facial_embedding`, `fingerprint_embedding`) | `mongodump --archive --gzip` | `mongo.archive.gz` |
| Redis 7 | Kong rate-limiting counters only — **not** backed up (rebuildable) | — | — |
| Kafka | Transient events — **not** backed up | — | — |

---

## Taking a backup (local / compose)

```bash
# Requires the stack running (docker compose up -d)
./database/scripts/backup.sh

# Custom destination
BACKUP_DIR=/mnt/backups ./database/scripts/backup.sh
```

Output lands in `database/backups/<timestamp>/` (or `BACKUP_DIR`).

---

## Restoring

```bash
# Recommended: the companion script (asks for confirmation unless FORCE=1)
./database/scripts/restore.sh database/backups/<timestamp>

# PostgreSQL manually (logical restore; --clean drops existing objects first)
pg_restore --clean -d faceattend_db faceattend_db.dump

# MongoDB manually
mongorestore --gzip --archive=mongo.archive.gz
```

> **Warning:** `docker compose down -v` deletes the `postgres_data` / `mongo_data` /
> `kafka_data` / `redis_data` volumes without prompting. Never use `-v` on a stack holding data
> you care about.

---

## Recovery targets

| Item | Target | Status |
|---|---|---|
| `faceattend_db` (prod, Neon) | Managed PITR by Neon | ✅ provider-managed |
| MongoDB embeddings (prod, EC2) | Manual `backup.sh` runs | ⚠️ no automation yet |
| Liquibase migrations | Re-runnable from scratch (`database/docker-compose.yml` chain) | ✅ idempotent, additive |
| Bootstrap/admin seed | `01-ms-identity-db/02-dml/001-bootstrap-admin-user.yaml` (idempotent `ON CONFLICT DO NOTHING`) | ✅ |

---

## Gaps (honest)

- No cron/scheduler for `backup.sh`; no off-host replication for MongoDB.
- No documented restore **drill** — `restore.sh` exists and its commands are listed above, but the procedure has not been rehearsed end-to-end.
- `NFR-010` (RTO/RPO) is **not yet met**: production is a single EC2 instance (ADR-020 risks).

---

## Correlations

- Deploy record & production topology → `FULL/DESPLIEGUE.md`, ADR-020
- Migrations (what must be re-applied after a restore) → `05-architecture/decisions/records/ADR-010-database-migrations.md`
- Incident response → `13-operations/incident-management.md`
- Local stack & volumes → `FULL/COMPOSE.md`
