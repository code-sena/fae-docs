# Events — Quality Service

> **No events at all — neither published nor consumed.** Verified against
> `FULL/back-end/09-ms-quality/src` on 2026-10-17.

---

## Events this service PUBLISHES

**None.** Despite its name, `src/infrastructure/messaging/event.publisher.ts` never touches a
broker: `registerEventHook` only registers a Fastify `onResponse` hook that writes a structured
log line for mutating requests under `/api/v1/quality/`:

```ts
// Best-effort Kafka hook (mismo patrón que academic/configuration): no bloquea el CRUD.
export function registerEventHook(app: FastifyInstance) {
  app.addHook('onResponse', async (req) => {
    if ((req.method === 'POST' || req.method === 'PUT' || req.method === 'PATCH' || req.method === 'DELETE') && req.url.includes('/api/v1/quality/')) {
      req.log.info({ url: req.url, method: req.method }, 'quality domain event (best-effort)');
    }
  });
}
```

There is no `kafkajs` import anywhere in the service (the comment says "same pattern as
academic/configuration" — those two services do publish; quality only logs).

## Events this service CONSUMES

**None.** No consumer, no subscription, no offset management.

---

## Delivery guarantees

Not applicable — the service has zero messaging traffic. "Best-effort" in its source comment refers
to logging, not to delivery semantics.

---

## Correlations

- Real publisher/consumer inventory → `../communication-patterns.md` §2
- Target catalog (including what quality *might* publish someday) → `../event-catalog.md`
- Relevant ADR → `05-architecture/decisions/records/ADR-006-message-broker.md` (target state)
