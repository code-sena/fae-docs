# Hexagonal Architecture (Ports & Adapters)

> Hexagonal architecture, proposed by Alistair Cockburn, organizes a service so that the
> **business domain is completely independent** of the surrounding technology.
> The database, the web framework, the message broker — all are interchangeable details.
> What matters is the business logic, which lives at the center.

> **Stack note:** The concepts in this document are valid for any language.
> The code examples and folder structure specific to your technology are in:
> - Node.js + TypeScript → [`_stacks/node-typescript.md`](../_stacks/node-typescript.md)
> - Java + Spring Boot → [`_stacks/java-spring.md`](../_stacks/java-spring.md)
> - Python + FastAPI → [`_stacks/python-fastapi.md`](../_stacks/python-fastapi.md)
> - (Go → [`_stacks/go.md`](../_stacks/go.md) is kept as a generic reference only; no FaceAttend
>   service uses Go since `08-ms-notification` moved to TypeScript — ADR-021)

---

## The problem it solves

```
❌ Traditional layered architecture:

  [HTTP Controller]
       ↓
  [Service]
       ↓
  [Repository]
       ↓
  [Database]

Problem: The "Service" mixes business logic with framework calls.
If you change the framework, you break the business. If you want to test the business,
you need to simulate the database.
```

```
✓ Hexagonal Architecture:

  [HTTP Controller]  [CLI]  [Test]  ← Primary Adapters (enter the hexagon)
          │            │      │
          └────────────┴──────┘
                       │
                 [Driving Port]  ← Interface that defines the domain's API
                       │
               ┌───────────────┐
               │               │
               │    DOMAIN     │  ← Pure business logic, no external dependencies
               │               │
               └───────────────┘
                       │
                 [Driven Port]  ← Interface the domain needs from the outside world
                       │
          ┌────────────┴──────┐
          │                   │
  [DB Adapter]  [Kafka Adapter]  ← Secondary Adapters (exit the hexagon)
```

---

## Folder structure

```
src/
├── domain/                          # The hexagon — no frameworks, no external dependencies
│   ├── [aggregate]/
│   │   ├── [Aggregate].ts           # Aggregate Root with invariants
│   │   ├── [Aggregate]Id.ts         # Value Object for the ID
│   │   ├── events/
│   │   │   └── [EventOccurred].ts   # Domain events
│   │   ├── services/
│   │   │   └── [DomainService].ts   # Logic that does not belong to any entity
│   │   └── ports/                   # Interfaces (ports) — abstract contracts
│   │       ├── in/
│   │       │   └── [UseCasePort].ts # Driving port: use case contract
│   │       └── out/
│   │           └── [RepoPort].ts    # Driven port: repository contract
│   └── shared/
│       └── value-objects/           # VOs shared between aggregates
│           ├── Email.ts
│           └── Money.ts
│
├── application/                     # Use cases — orchestrate the domain
│   └── [aggregate]/
│       ├── [CreateXxxUseCase].ts    # Implements the driving port
│       └── dtos/
│           ├── [CreateXxxRequest].ts
│           └── [CreateXxxResponse].ts
│
├── infrastructure/                  # Everything external to the hexagon
│   ├── adapters/
│   │   ├── in/                      # Primary adapters — receive external calls
│   │   │   ├── http/
│   │   │   │   ├── [XxxController].ts
│   │   │   │   └── [XxxRouter].ts
│   │   │   └── messaging/
│   │   │       └── [XxxEventConsumer].ts
│   │   └── out/                     # Secondary adapters — call the outside
│   │       ├── persistence/
│   │       │   └── [XxxRepositoryImpl].ts   # Implements the driven port
│   │       ├── messaging/
│   │       │   └── [XxxEventPublisher].ts
│   │       └── external/
│   │           └── [ExternalApiAdapter].ts
│   └── config/
│       ├── database.ts
│       └── container.ts             # Dependency injection (IoC)
│
└── main.ts                          # Bootstrap — connects adapters with ports
```

---

## The Ports

Ports are **interfaces** (abstract contracts). The domain defines them;
adapters implement them.

### Driving Port (Input Port)

Defines what the domain can do — its public API from the outside's perspective.

```typescript
// src/domain/attendance/ports/in/RecordAttendancePort.ts
export interface RecordAttendancePort {
  execute(request: RecordAttendanceRequest): Promise<RecordAttendanceResponse>;
}
// Example FaceAttend aggregate: Attendance_Record enforces RN-22 (one per actor per session) and RN-23 (Open session)
```

### Driven Port (Output Port)

Defines what the domain needs from the outside world — without knowing how it is implemented.

```typescript
// src/domain/attendance/ports/out/AttendanceRepositoryPort.ts
export interface AttendanceRepositoryPort {
  save(record: AttendanceRecord): Promise<void>;
  findBySessionAndActor(sessionId: ClassSessionId, actorId: AcademicActorId): Promise<AttendanceRecord | null>;
}

// src/domain/order/ports/out/EventPublisherPort.ts
export interface EventPublisherPort {
  publish(event: DomainEvent): Promise<void>;
}
```

---

## The Adapters

### Primary Adapter — HTTP Controller

The HTTP controller translates the HTTP request to the domain use case.

```typescript
// src/infrastructure/adapters/in/http/OrderController.ts
import { CreateOrderPort } from '@domain/order/ports/in/CreateOrderPort';

@Controller('/orders')
export class OrderController {
  constructor(
    // Inject the port, NOT the concrete implementation
    private readonly createOrder: CreateOrderPort,
  ) {}

  @Post('/')
  async create(@Body() body: CreateOrderHttpRequest): Promise<void> {
    // Translate HTTP request → domain DTO
    const request = new CreateOrderRequest(body.customerId, body.items);
    // Call the use case through the port
    const response = await this.createOrder.execute(request);
    return response;
  }
}
```

### Secondary Adapter — Repository

The repository implements the driven port. The domain does not know PostgreSQL exists.

```typescript
// src/infrastructure/adapters/out/persistence/OrderRepositoryImpl.ts
import { OrderRepositoryPort } from '@domain/order/ports/out/OrderRepositoryPort';

export class OrderRepositoryImpl implements OrderRepositoryPort {
  constructor(private readonly db: DatabaseConnection) {}

  async save(order: Order): Promise<void> {
    // Translate Aggregate → database row
    await this.db.query(
      'INSERT INTO orders (id, customer_id, status, total) VALUES ($1, $2, $3, $4)',
      [order.id.value, order.customerId.value, order.status, order.total.amount],
    );
  }

  async findById(id: OrderId): Promise<Order | null> {
    const row = await this.db.queryOne('SELECT * FROM orders WHERE id = $1', [id.value]);
    if (!row) return null;
    // Translate database row → Aggregate
    return OrderMapper.toDomain(row);
  }
}
```

---

## The Use Case (Application Service)

The use case orchestrates the domain. It uses driving and driven ports. It contains no business logic — that lives in the Aggregate.

```typescript
// src/application/attendance/RecordAttendanceUseCase.ts
import { RecordAttendancePort } from '@domain/attendance/ports/in/RecordAttendancePort';
import { AttendanceRepositoryPort } from '@domain/attendance/ports/out/AttendanceRepositoryPort';
import { EventPublisherPort } from '@domain/order/ports/out/EventPublisherPort';

export class RecordAttendanceUseCase implements RecordAttendancePort {
  constructor(
    private readonly attendanceRepo: AttendanceRepositoryPort,
    private readonly eventPublisher: EventPublisherPort,
  ) {}

  async execute(request: RecordAttendanceRequest): Promise<RecordAttendanceResponse> {
    // 1. Create aggregate — enforces RN-22/RN-23 in domain
    const record = AttendanceRecord.create(request.sessionId, request.actorId, request.captureMethod);

    // 2. Persist via port (domain unaware of PG)
    await this.attendanceRepo.save(record);

    // 3. Publish AttendanceRecorded
    for (const event of record.domainEvents) {
      await this.eventPublisher.publish(event);
    }

    return new RecordAttendanceResponse(record.id.value);
  }
}
```

---

## The Dependency Rule

> **Dependencies always point inward.**
> The domain does not import anything from application or infrastructure.
> Infrastructure imports from the domain (but never the other way around).

```
infrastructure/ → application/ → domain/
                                    ↑
                         CANNOT import anything from application/ or infrastructure/
```

### Dependency inversion (DI) in practice

```typescript
// ✓ Correct — domain defines the interface, infrastructure implements it
// In domain/:
export interface OrderRepositoryPort { ... }

// In infrastructure/:
export class OrderRepositoryImpl implements OrderRepositoryPort { ... }

// In the bootstrap (main.ts), the concrete implementation is injected:
const orderRepo = new OrderRepositoryImpl(dbConnection);
const createOrderUseCase = new CreateOrderUseCase(orderRepo, eventPublisher);
const orderController = new OrderController(createOrderUseCase);
```

---

## Advantages for TDD

Hexagonal architecture is ideal for TDD because:

1. **The domain is testable without framework mocks.** You do not need to start a server
   or a database to test business logic.

2. **Driven ports can be faked easily.** In tests, you use an
   in-memory repository (Fake) instead of the real one.

3. **Invariants are explicit** and tested in isolation.

```typescript
// Domain unit test — zero external dependencies
describe('Order', () => {
  it('cannot be created without items', () => {
    expect(() => Order.create(customerId, [])).toThrow('INV-001');
  });

  it('on confirm changes status to CONFIRMED', () => {
    const order = Order.create(customerId, [validItem]);
    order.confirm();
    expect(order.status).toBe(OrderStatus.CONFIRMED);
  });

  it('on confirm emits OrderConfirmed event', () => {
    const order = Order.create(customerId, [validItem]);
    order.confirm();
    expect(order.domainEvents).toContainEqual(expect.any(OrderConfirmedEvent));
  });
});

// Use case test with FAKE repository (not a real DB mock)
describe('CreateOrderUseCase', () => {
  it('saves the order and publishes the event', async () => {
    const fakeOrderRepo = new InMemoryOrderRepository();
    const fakeEventPublisher = new InMemoryEventPublisher();
    const useCase = new CreateOrderUseCase(fakeOrderRepo, fakeEventPublisher);

    await useCase.execute(new CreateOrderRequest(customerId, [validItem]));

    expect(fakeOrderRepo.orders).toHaveLength(1);
    expect(fakeEventPublisher.events).toContainEqual(expect.any(OrderCreated));
  });
});
```

> See full TDD guide in `11-quality/tdd-guide.md`

---

## Hexagonal Architecture Checklist

When reviewing a PR or new service, verify:

- [ ] `domain/` has no imports from `infrastructure/` or `application/`
- [ ] `domain/` has no imports from frameworks (Express, NestJS, TypeORM, etc.)
- [ ] Every repository interface lives in `domain/ports/out/`
- [ ] Every use case interface lives in `domain/ports/in/`
- [ ] Mappers (`toDomain` / `toPersistence`) live in `infrastructure/`, not in `domain/`
- [ ] HTTP API DTOs live in `infrastructure/adapters/in/http/`, not in `domain/`
- [ ] There is a unit test for each Aggregate invariant

---

## Common mistakes (anti-patterns)

| Anti-pattern | Why it is bad | Solution |
|-------------|--------------|---------|
| `import { Repository } from 'typeorm'` in the domain | Couples the domain to TypeORM | Define your own port interface |
| Business logic in the Controller | If you change the endpoint, you change the business | Move to the Aggregate |
| Repository returning DTOs instead of Aggregates | The domain cannot validate invariants | Use Mapper to reconstruct the Aggregate |
| Use case with 15 dependencies | It probably does too much | Split into smaller use cases |
| `any` in port interfaces | You lose the typed contract | Always use explicit typing |

---

## References and correlations

- Bounded Contexts → `02-domain/domain-map.md`
- Entities and invariants → `02-domain/entities-and-rules.md`
- Domain events → `02-domain/domain-events.md`
- Complementary patterns (CQRS, Event Sourcing, Saga) → `05-architecture/pattern-guide.md`
- TDD applied to hexagonal architecture → `11-quality/tdd-guide.md`
- Service template with hexagonal structure → `09-microservices/_template/service/`
