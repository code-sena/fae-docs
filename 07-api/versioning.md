# API Versioning — FaceAttendEDU

> URL versioning is the only supported strategy. Real prefix in all services and Kong: `/api/v1/`.

## 1. Rule

- Canonical: `/api/v1/<resource>` via Kong `:8000`.
- Direct aliases without prefix (`/roles`, `/alerts`, …) exist for local dev only. Never use in staging/prod, never add new ones.
- Servers in every OpenAPI contract:
  - `http://localhost:<port>` (direct, dev)
  - `http://localhost:8000` (via Kong, dev — preferred for frontends)
  - `https://api.dev.{domain}` / `https://api.{domain}` (staging/prod via Kong)

## 2. What is breaking vs non-breaking

Breaking (requires `/api/v2/` alongside v1):
- Remove/rename path, param, or field; change type/format; tighten validation; change auth requirement from public→JWT; change success code.

Non-breaking (same version):
- Add optional field/filter/nested read; add new path; loosen validation; add new error code (documented).

## 3. Deprecation policy

> Decisions 2026-10-03: `cities` and `persons/{id}/transfer` stay deprecated; canonical deletion targets are `CityController`, Kong `/cities` route and Mobile `CityService`. Single DB `faceattend_db` stays canonical dev; `init-multidb.sql` is alt-prod only.

1. Announce in `endpoint-matrix.md` + contract `deprecated: true` + `Sunset` header.
2. Keep v1 working for ≥1 release / ≥30 days.
3. Gateway keeps routing both versions; monitor `x-total-count` consumers (Mobile uses `_limit`, must migrate to `limit/offset`).
4. Remove only after frontends confirm migration (`frontend-integration.md` updated).

## 4. Current version

All 9 services + gateway are `1.0.0` (`info.version` in every `contracts/openapi/*.yaml`). No v2 exists yet.

## 5. Examples

```
# v1 canonical (via gateway)
GET http://localhost:8000/api/v1/attendance-records?classSessionId=12

# direct (dev only)
GET http://localhost:8085/api/v1/attendance-records?classSessionId=12

# future v2 (additive major)
GET http://localhost:8000/api/v2/attendance-records?classSessionId=12
```
