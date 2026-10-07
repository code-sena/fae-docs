# Example: C4 Container Diagram

> This file shows how to document the system with a C4 Level 2 (Container) diagram.
> The diagram uses Mermaid, which renders natively on GitHub, GitLab and most
> modern editors (VS Code with extension, Obsidian, etc.).
>
> **Instruction:** Copy this structure, replace the example services (api-gateway,
> identity-service) with your project's real services, and move it to `diagrams/source/c4-container.md`.

---

## Example system: [X] management platform

```mermaid
C4Container
    title Container Diagram — [System Name]

    Person(user, "User / Client", "Accesses the system from browser or mobile app")
    Person(admin, "Administrator", "Manages system configuration and users")

    System_Boundary(system, "[System Name]") {

        Container(frontend, "Web Frontend", "React / Vue / Angular", "User interface. SPA served as static files.")

        Container(gateway, "API Gateway", "[Your framework]", "Single entry point. Authenticates JWT tokens, routes to the right service, applies rate limiting.")

        Container(auth, "Identity Service", "[Your stack]", "Manages people, users, authentication (login/register/refresh). Owns Person and User entities.")

        Container(service_a, "[Service A]", "[Your stack]", "[Main responsibility of Service A. Owns [Entity A].]")

        Container(service_b, "[Service B]", "[Your stack]", "[Main responsibility of Service B. Owns [Entity B].]")

        ContainerDb(db_auth, "Identity DB", "PostgreSQL", "People, users, sessions, tokens.")
        ContainerDb(db_a, "[Service A] DB", "[Chosen engine]", "[Service A] domain data.")
        ContainerDb(db_b, "[Service B] DB", "[Chosen engine]", "[Service B] domain data.")
        ContainerDb(redis, "Redis", "Redis", "Token blacklist, rate limiting, shared cache.")
        ContainerDb(broker, "Message Broker", "Kafka (KRaft)", "Domain events between services.")
    }

    System_Ext(ext_email, "Email Service", "Sendgrid / SES / SMTP")
    System_Ext(ext_payment, "Payment Gateway", "[If it applies to the project]")

    %% User → System relationships
    Rel(user, frontend, "Uses", "HTTPS")
    Rel(admin, frontend, "Manages", "HTTPS")
    Rel(frontend, gateway, "API calls", "HTTPS / REST")

    %% Gateway → Services
    Rel(gateway, auth, "Verifies tokens / routes", "Internal HTTP")
    Rel(gateway, service_a, "Routes requests", "Internal HTTP")
    Rel(gateway, service_b, "Routes requests", "Internal HTTP")

    %% Services → DB (each service owns its DB)
    Rel(auth, db_auth, "Reads / writes", "SQL")
    Rel(auth, redis, "Token blacklist", "Redis protocol")
    Rel(gateway, redis, "Rate limiting", "Redis protocol")
    Rel(service_a, db_a, "Reads / writes", "SQL / NoSQL")
    Rel(service_b, db_b, "Reads / writes", "SQL / NoSQL")

    %% Async communication
    Rel(auth, broker, "Publishes events (user.registered)", "Async")
    Rel(service_a, broker, "Publishes / consumes events", "Async")
    Rel(service_b, broker, "Consumes [Service A] events", "Async")

    %% External systems
    Rel(service_b, ext_email, "Sends notifications", "HTTPS / SMTP")
    Rel(service_a, ext_payment, "Processes payments", "HTTPS")
```

---

## How to use this diagram

1. **Render on GitHub:** The diagram renders automatically on push. No extra tooling needed.

2. **Render in VS Code:** Install the "Markdown Preview Mermaid Support" extension or use the official Mermaid extension.

3. **Export as image:** Use the [Mermaid CLI](https://github.com/mermaid-js/mermaid-cli):
```bash
mmdc -i c4-container-example.md -o ../exports/c4-container.svg
```

4. **Update the diagram:** The diagram lives in Git with the code. When you change the architecture (new service, new DB engine), update the diagram in the same PR.

---

## Conventions for this project

| Element | Color / Style | When to use it |
|----------|---------------|----------------|
| `Container` | Blue | Deployable microservice or app |
| `ContainerDb` | Blue cylinder | Database, cache, broker |
| `Person` | Person icon | Human actor |
| `System_Ext` | Gray | Third-party system you do not control |

---

## Correlations

- Service catalog → `09-microservices/service-catalog.md`
- Each service detail → `09-microservices/services/`
- All diagrams index → `08-uml/diagram-index.md`
