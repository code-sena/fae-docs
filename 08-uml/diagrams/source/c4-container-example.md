# Template: C4 Container diagram

> **Status:** template only — the real diagram for this project already exists at
> [`c2-container-diagram.mmd`](./c2-container-diagram.mmd) (registered as C4-02 in
> [`../../diagram-index.md`](../../diagram-index.md)). **Do not copy this file to create a new
> diagram**: create a new `.mmd` file in `diagrams/source/` and add a row to the registry.

---

## How to add a new C4 diagram

1. Create `diagrams/source/<name>.mmd` (see the naming rules in `../../diagram-index.md`).
2. Start from the skeleton below — one diagram, one file, Mermaid only.
3. Preview it in VS Code (Mermaid extension) or on [mermaid.live](https://mermaid.live).
   GitHub does **not** render standalone `.mmd` files; export to SVG (step 5) if it must be visible there.
4. Register it in `../../diagram-index.md` with ID, type and status.
5. Only if needed: `npx.cmd -y @mermaid-js/mermaid-cli -i diagrams/source/<name>.mmd -o diagrams/exports/<name>.svg`

## Skeleton (C4 Level 2)

```mermaid
C4Container
    title Container Diagram — [System Name]

    Person(user, "User", "Who uses the system and from which client")

    System_Boundary(system, "[System Name]") {

        Container(web, "Web App", "[stack]", "Browser client")
        Container(gateway, "API Gateway", "[gateway]", "Single entry point: routing, CORS, rate limiting")

        Container(service_a, "[Service A]", "[stack]", "[Responsibility. Owns [Entity A].]")
        Container(service_b, "[Service B]", "[stack]", "[Responsibility. Owns [Entity B].]")

        ContainerDb(db_a, "[Service A] DB", "[engine]", "[Data owned by A]")
        ContainerDb(db_b, "[Service B] DB", "[engine]", "[Data owned by B]")
        ContainerDb(broker, "Message Broker", "[broker]", "Domain events")
    }

    System_Ext(ext, "[External system]", "[What it does and how you call it]")

    Rel(user, web, "Uses", "HTTPS")
    Rel(web, gateway, "API calls", "HTTPS / REST")
    Rel(gateway, service_a, "Routes", "HTTP / REST")
    Rel(gateway, service_b, "Routes", "HTTP / REST")
    Rel(service_a, db_a, "Reads / writes", "SQL")
    Rel(service_b, db_b, "Reads / writes", "SQL")
    Rel(service_a, broker, "Publishes events", "Async")
    Rel(service_b, broker, "Consumes events", "Async")
    Rel(service_a, ext, "Calls", "HTTPS")
```

## Conventions for this project

| Element | When to use it |
|---------|----------------|
| `Container` | Deployable microservice or app (one per service — see `09-microservices/service-catalog.md`) |
| `ContainerDb` | Database, cache or broker |
| `Person` | Human actor (student, instructor, administrator) |
| `System_Ext` | Third-party system you do not control (e.g. `countriesnow.space`, SMTP) |

**Accuracy rules (this repo):**
- Kong authenticates **at the edge**: only `cors` + `rate-limiting` plugins and a global
  `pre-function` (opaque `sessionId` introspected against identity, no `jwt`, no Redis token
  blacklist — Redis holds rate-limit counters only); the Java services check the same
  `sessionId` again (plus RBAC) — see `07-api/authentication.md`.
- No payment gateway exists in this system; do not add one to a diagram.
- Ports, stacks and DB engines must match `09-microservices/service-catalog.md`.

---

## Correlations

- Service catalog → `09-microservices/service-catalog.md`
- Each service detail → `09-microservices/services/`
- All diagrams index → `08-uml/diagram-index.md`
