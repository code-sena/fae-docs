# ADR-015 — Caching Strategy

- **ID:** ADR-015
- **Date:** 2026-09-12
- **Status:** Proposed
- **Authors:** FaceAttend EDU Team

---

## Context

Redis is already used for session storage (identity-service, api-gateway per `service-catalog.md`). The system must decide what else to cache, which caching patterns to use, and how to manage cache lifecycle.

Caching is relevant for:
1. Session tokens (already decided — Redis).
2. Permission/role lookups (called on every request via authorization-service).
3. Configuration values (change infrequently, read frequently).
4. Academic entity lookups (student/teacher profiles, course data).
5. API Gateway response caching for read-heavy endpoints.

**Known constraints:**
- ADR-002 establishes a modular monolith — all modules share a single Redis instance.
- NFR-001 requires P95 < 2000ms for critical endpoints — caching can help achieve this.
- NFR-004 requires sensitive data to be handled carefully — cached data must respect access controls.
- Team chose ioredis as the Redis client (direct, no abstraction layer).

---

## Decision

**We decided:** Use **Redis** as the shared cache, with **ioredis** as the client library. Caching is applied selectively per data type with specific TTLs and invalidation strategies.

### What to cache and how

| Data type | Cache pattern | TTL | Invalidation | Rationale |
|-----------|--------------|-----|-------------|-----------|
| **Session tokens** | Write-through | 7 days (refresh token) / 1 hour (access token) | Explicit delete on logout | Already decided; sessions must be fast |
| **Permission/role lookups** | Cache-aside | 5 minutes | On `UserRoleGranted` / `UserRoleRevoked` event | Called on every request; permissions change rarely |
| **Academic configuration** | Cache-aside | 15 minutes | On `ConfigurationUpdated` event | Read-heavy; changes are infrequent admin actions |
| **Student/Teacher profiles** | Cache-aside | 10 minutes | On `PersonCreated` / `PersonUpdated` event | Read during attendance; write is rare |
| **Course/Enrollment lists** | Cache-aside | 5 minutes | On `EnrollmentCreated` / `CourseDeleted` event | Read during schedule queries; writes are periodic |

### What NOT to cache

| Data type | Reason |
|-----------|--------|
| Attendance records | Write-once, read-for-reports; freshness required; low read volume |
| Biometric embeddings | Already in MongoDB with vector search index; caching adds complexity |
| Audit events | Append-only; read is rare and time-range based |
| Justification documents | Stored in MinIO (ADR-009); binary data not suitable for Redis |

### Cache key naming convention

```
{context}:{entity}:{id}

Examples:
  authorization:permissions:{userId}        → Set of permission strings
  configuration:academic:{schoolId}         → Academic configuration object
  identity:person:{personId}               → Person profile
  identity:session:{sessionId}             → Session data
```

### Cache-aside implementation pattern

```typescript
// Cache-aside pattern with ioredis
async function getStudentProfile(studentId: string): Promise<StudentProfile> {
  const cacheKey = `identity:person:${studentId}`;

  // 1. Try cache
  const cached = await redis.get(cacheKey);
  if (cached) {
    return JSON.parse(cached);
  }

  // 2. Cache miss — query database
  const profile = await personRepository.findById(studentId);
  if (profile) {
    // 3. Populate cache with TTL
    await redis.set(cacheKey, JSON.stringify(profile), 'EX', 600); // 10 min
  }

  return profile;
}

// Invalidation on domain event
async function onPersonUpdated(event: PersonUpdated): Promise<void> {
  const cacheKey = `identity:person:${event.personId}`;
  await redis.del(cacheKey);
}
```

---

## Evaluated alternatives

| Alternative | Pros | Cons | Reason for discarding |
|------------|------|------|-----------------------|
| **Redis + ioredis cache-aside (chosen)** | Already in use for sessions; fast; ioredis supports all Redis data structures; simple to implement | Single Redis instance for all modules (ADR-002); cache invalidation is manual | — (chosen) |
| cache-manager + ioredis | Abstraction layer; automatic TTLs; multi-tier support | Adds overhead; less control over cache behavior; team chose ioredis direct | Overhead not justified for selective caching |
| In-memory cache (Map/LRU) | Zero infrastructure; fastest possible | Not shared across process restarts; not shared across modules if extracted; no persistence | Not suitable for modular monolith with shared state |
| PostgreSQL query cache | No additional infrastructure | PostgreSQL is not designed for caching; adds load to the database | Wrong tool; Redis is purpose-built for caching |
| CDN caching (Cloudflare, etc.) | Edge caching; reduces server load | Only for static assets and public APIs; not suitable for authenticated dynamic data | Most data requires authentication; CDN is inappropriate |

---

## Consequences

**Positive:**
- Permission lookups (called on every request) are served from cache with <1ms latency.
- Configuration and profile reads are faster, improving P95 for critical endpoints.
- Cache-aside pattern is simple to implement and reason about.
- Event-driven invalidation keeps cache consistent with database state.

**Negative / Trade-offs:**
- Cache invalidation is a known hard problem — stale data is possible during the TTL window.
- Redis is a single instance shared by all modules (ADR-002 constraint) — one module's heavy caching can affect others.
- Cache-aside requires two reads on miss (cache + DB) — adds complexity.

**Impact on the system:**
- Affected modules: all modules that read cross-context data.
- Documents that must be updated:
  - `09-microservices/service-catalog.md` — document Redis usage per service
  - `.env.example` — confirm `APP_*_REDIS_URL` variables per service
  - `09-microservices/communication-patterns.md` — document cache invalidation via events
  - `10-devops/local-setup.md` — confirm Redis in Docker Compose

---

## Risks

| Risk | Probability | Impact | Mitigation |
|------|------------|--------|-----------|
| Stale cache serves incorrect permissions | Low | High | Short TTL (5 min) for permissions; explicit invalidation on role changes; worst case: user sees stale permissions for 5 min |
| Redis failure loses all cached data | Low | Medium | Cache-aside means DB is always the source of truth; Redis restart repopulates cache naturally |
| Cache stampede on Redis restart | Medium | Medium | Use jittered TTLs (random ±30s); limit concurrent DB queries with mutex/lock |

---

## References

- Service catalog (Redis usage) → `09-microservices/service-catalog.md`
- Communication patterns (event-driven invalidation) → `09-microservices/communication-patterns.md`
- NFR-001 Performance → `04-requirements/non-functional.md`
- Related to: ADR-002 (single Redis instance), ADR-008 (session tokens)
