# Quality Service (tooling — not a bounded context)

> Serves the ISO 25010 / ISO 29110 / ISTQB quality metrics used by the project's quality
> reports. It owns **no business entity** and is not one of the 8 DDD bounded contexts.

---

## Responsibility

> Computes and exposes quality indicators over its own `quality` schema
> (`quality_project`, `quality_evaluation`, `quality_evaluation_item`, `process_assessment`,
> `process_assessment_rating`, `istqb_assessment`, `istqb_assessment_item`). Owned by
> `11-quality`, not by a product team.

---

## Location in the architecture

| Field | Value |
|-------|-------|
| Number in catalog | 09 |
| Local port | 8089 |
| Schema | `quality` (PostgreSQL 17, created by `07-ms-configuration-db`) |
| Stack | TypeScript + Fastify + Zod + pino |
| Folder in repo | `FULL/back-end/09-ms-quality/` |
| Communicates with | nothing downstream; Kong exposes `/api/v1/quality` (rate limit 100/min) and the gateway health route `GET /health/quality` is proxied to `ms-academic:8083` |
| Consumed by | quality dashboards and the gateway health check |

---

## Related documents

- [API Contract](../../../07-api/contracts/openapi/09-quality-service.yaml)
- Port matrix → [`../../07-api/README.md`](../../07-api/README.md)
- Per-service pages (`data-model.md`, `events.md`, `decisions.md`, `runbook.md`) are **pending** —
  follow `../_template/service/` when writing them.
