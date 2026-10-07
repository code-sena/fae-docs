# ADR-010 — Database Migration Strategy

- **ID:** ADR-010
- **Date:** 2026-09-12
- **Status:** Proposed
- **Implementation note (2026-10-05, verified against `FULL/`):** Liquibase is the only migration tool — image `liquibase/liquibase:4.29.0`, **8 chained jobs** (`identity → authorization → academic → scheduling → attendance → biometric → configuration → notification`) run as containers from `database/docker-compose.yml`, sequenced with `service_completed_successfully`. Changelogs are **YAML changesets embedding SQL** under `database/<NN>-ms-*-db/changelog/` (not standalone `.sql` files), and there is **no Flyway** anywhere in the repo.
- **Authors:** FaceAttend EDU Team

---

## Context

The system uses a single PostgreSQL instance with schema-per-context (ADR-002), plus MongoDB for the biometric context (ADR-003). Schema changes must be:

1. Repeatable across environments (local, dev, staging, production — per `10-devops/environments.md`).
2. Version-controlled and reviewable.
3. Reversible (NFR-009 requires rollback capability; `environments.md` references `npm run db:migrate:revert`).
4. Compatible with multiple schemas (one per bounded context) in the same PostgreSQL instance.
5. Capable of handling both PostgreSQL and MongoDB schema changes.

**Known constraints:**
- ADR-002 establishes 7 PostgreSQL schemas in one `faceattend_db` instance plus MongoDB for biometric embeddings (with `biometric_update_case` hosted in `configuration`).
- Team of 3 with no dedicated DBA.
- `environments.md` CI/CD pipeline expects repeatable migrations.
- The modular monolith means all schemas are managed by a single application process.

---

## Decision

**We decided:** Use **Liquibase** as the database migration tool for PostgreSQL schemas, with change logs stored as SQL files.

**Justification:**

1. **Multi-schema support:** Liquibase natively supports targeting changes to specific schemas (`objectQuotingScheme="QUOTE_ALL_OBJECTS"` + schema parameter), which maps directly to ADR-002's schema-per-context model.
2. **Database-agnostic:** Liquibase works with PostgreSQL (and could handle other databases if needed), providing a consistent migration tool across the stack.
3. **SQL-native:** Change logs are written as raw SQL (not XML or YAML), making them readable by anyone who knows SQL. This lowers the learning curve for the team.
4. **Rollback support:** Liquibase supports `rollback` commands for forward-only migrations, satisfying the revert requirement in `environments.md`.
5. **CLI + library:** Liquibase can run as a CLI tool (in Jenkins pipelines) or as a Java library (in application startup).
6. **Integration with CI/CD:** Liquibase integrates naturally with Jenkins pipelines via CLI.

**Implementation:**
- Each bounded context has its own Liquibase changelog directory: `db/changelog/{context-name}/`.
- Change logs are SQL files with sequential numbering: `001_create_tables.sql`, `002_add_indexes.sql`.
- A master changelog includes all context changelogs in dependency order.
- Liquibase runs in the Jenkins pipeline before application deploy (per `environments.md`: "if there are database changes: migration ran before the code deploy").
- MongoDB schema changes (biometric) are handled separately via migration scripts (not Liquibase), since MongoDB is schemaless.

**Migration file structure:**
```
db/
├── changelog/
│   ├── master.xml                 (includes all context changelogs)
│   ├── identity/
│   │   ├── 001_create_person.sql
│   │   ├── 002_create_user.sql
│   │   └── 003_create_session.sql
│   ├── authorization/
│   │   ├── 001_create_role.sql
│   │   └── 002_create_permission.sql
│   ├── academic/
│   │   ├── 001_create_school.sql
│   │   └── ...
│   ├── scheduling/
│   │   └── ...
│   ├── attendance/
│   │   └── ...
│   ├── notification/
│   │   └── ...
│   └── configuration/
│       └── ...
```

---

## Evaluated alternatives

| Alternative | Pros | Cons | Reason for discarding |
|------------|------|------|-----------------------|
| **Liquibase with SQL changelogs (chosen)** | Multi-schema support; SQL-native; rollback support; CLI for CI/CD; database-agnostic | Requires Java runtime for CLI; XML master changelog is verbose | — (chosen) |
| TypeORM Migrations | TypeScript-native; integrated with the ORM; auto-generates migrations from entity changes | Tightly coupled to TypeORM; Python modules cannot use it; rollback is limited; multi-schema requires manual configuration | Cannot be used by Python biometric-service; coupled to one ORM |
| Knex.js Migrations | JavaScript-native; simple API; good for single-schema | Limited multi-schema support; coupled to Knex query builder; cannot be used by Python modules | Not designed for multi-schema; cannot serve the Python biometric-service |
| Prisma Migrate | Modern; schema-first; auto-generates migrations | Schemaless for MongoDB; tightly coupled to Prisma ORM; no SQL-native changelogs; less mature rollback | Coupled to Prisma; limited multi-schema support; cannot serve Python modules |
| Flyway | SQL-native; widely used; simple | Requires Java runtime; less multi-schema support than Liquibase; limited rollback (paid feature in some editions) | Liquibase is more capable for multi-schema; rollback is free in Liquibase |
| Alembic (Python) | Python-native for biometric-service | Cannot manage PostgreSQL schemas from TypeScript modules; separate tool for each language | Would require two migration tools; Liquibase handles both |

---

## Consequences

**Positive:**
- Single migration tool for all 7 PostgreSQL schemas.
- SQL-native changelogs are readable and version-controllable.
- CLI integration with Jenkins pipelines is straightforward.
- Rollback support satisfies `environments.md` revert requirement.

**Negative / Trade-offs:**
- Liquibase requires a Java runtime (JRE) for CLI execution — adds a dependency to the CI/CD pipeline.
- The XML master changelog is verbose (but individual change sets are SQL).
- MongoDB schema changes must be handled separately (no Liquibase support for MongoDB collections).

**Impact on the system:**
- Affected modules: all 7 PostgreSQL-backed modules.
- Documents that must be updated:
  - `10-devops/local-setup.md` — document Liquibase CLI installation
  - `10-devops/environments.md` — confirm Liquibase as the migration tool in pipeline steps
  - `.env.example` — add `LIQUIBASE_COMMAND` or equivalent configuration
  - `09-microservices/service-catalog.md` — reference migration tool per service

---

## Risks

| Risk | Probability | Impact | Mitigation |
|------|------------|--------|-----------|
| Java runtime dependency complicates CI/CD | Low | Low | Jenkins agents typically have Java installed; document requirement in `10-devops/` |
| Liquibase changelog conflicts between modules | Medium | Medium | Each module owns its changelog directory; no shared tables across schemas (ADR-002) prevents conflicts |
| MongoDB schema changes are not managed by Liquibase | Low | Low | MongoDB is schemaless; document migration patterns for biometric collections in `09-microservices/services/06-biometric-service/` |

---

## References

- ADR-002 (schema-per-context strategy) → `05-architecture/decisions/records/ADR-002-modular-monolith.md`
- ADR-003 (MongoDB for biometric) → `05-architecture/decisions/records/ADR-003-biometric-persistence.md`
- CI/CD pipeline → `10-devops/environments.md`
- Local setup → `10-devops/local-setup.md`
