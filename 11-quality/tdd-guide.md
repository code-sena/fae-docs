# TDD Guide — Test-Driven Development

> TDD is not about testing — it is about **design**. Writing the test first forces you to think
> about the interface before the implementation. The result: simpler, more decoupled code
> with a test suite that documents system behavior.

> **Stack note:** The TDD principles and cycles described here apply to any language.
> FaceAttend EDU is polyglot; concrete examples below are annotated per stack.
> - Node.js + TypeScript → [`_stacks/node-typescript.md`](../_stacks/node-typescript.md) (03-academic, 07-configuration, 08-notification, 09-quality — Fastify)
> - Java + Spring Boot → [`_stacks/java-spring.md`](../_stacks/java-spring.md) (01-identity, 02-authorization, 04-scheduling, 05-attendance)
> - Python + FastAPI → [`_stacks/python-fastapi.md`](../_stacks/python-fastapi.md) (06-biometric)

**Repo evidence:** every existing test in this project follows the cycle documented here —
`UserActivationServiceTest` (domain), `ResetPasswordUseCaseImplTest` (application, pure Fakes),
`sender.test.ts` (jest spy on the nodemailer transport), `test_router_contract.py` (contract tier). IEEE 829
test IDs (`TC-xx-xxx`) appear in test names so each test traces back to a requirement.

---

## The Red-Green-Refactor cycle

```
        ┌──────────────────────────────────────────────────────────┐
        │                                                          │
        ▼                                                          │
   ┌─────────┐                                                     │
   │   RED   │  Write the smallest test that can fail.             │
   │  🔴     │  Do NOT implement anything yet.                     │
   └────┬────┘  The test must fail for the right reason.           │
        │                                                          │
        ▼                                                          │
   ┌─────────┐                                                     │
   │  GREEN  │  Write the MINIMUM code to make the test pass.      │
   │  🟢     │  Do not aim for elegance here. Just make it pass.   │
   └────┬────┘                                                     │
        │                                                          │
        ▼                                                          │
   ┌──────────────┐                                                 │
   │   REFACTOR   │  Improve code without changing behavior.       │
   │  ♻️          │  Tests must remain green.                      │
   └──────────────┘                                                 │
        │                                                          │
        └──────────────────────────────────────────────────────────┘
```

**The rule of 3 moments:**
1. `RED`: The test fails — confirms the test can detect the bug
2. `GREEN`: The test passes — the code does the bare minimum needed
3. `REFACTOR`: The code is clean — no duplication, well named

**Commands per stack (repo defaults):**

| Stack | Run one test | Watch mode |
|-------|--------------|------------|
| Java (Maven) | `mvn test -Dtest=UserActivationServiceTest#shouldThrowExceptionWhenPersonIsInactive` | `mvn test` after each edit (IDE) |
| TS/Fastify | `npm run test -- academic.test.ts` (vitest/jest) | `npm run test:watch` |
| Python/FastAPI | `poetry run pytest tests/unit/web/test_router_contract.py -k enroll` | `pytest-watch` |

---

## The 3 testing principles (FIRST)

Good tests are:

| Letter | Principle | Description | Repo check |
|--------|-----------|-------------|------------|
| **F** | Fast | Run in milliseconds, not seconds | Unit suites (identity, notification) run < 1 s |
| **I** | Isolated | Do not depend on other tests or execution order | `@BeforeEach` / `beforeEach()` rebuild doubles fresh |
| **R** | Repeatable | Same result every time, regardless of environment | No network: Mongo disabled in contract tests, SMTP faked via jest mock of nodemailer |
| **S** | Self-validating | Pass / Fail without manual interpretation | AssertJ / expect / testify assertions only |
| **T** | Timely | Written BEFORE the code, not after | Required for new features; retro-applied via IEEE 829 suites |

---

## Test Doubles: the complete taxonomy

When the domain needs external collaborators (repositories, APIs), we replace them
in tests with doubles. Not all doubles are the same. In this repo the ports are explicit:
Java services define `application/port/out/*` interfaces; biometric overrides FastAPI
dependencies (`get_facial_repository`, …); notification injects an `email.Sender` interface.

### 1. Dummy
Does nothing. Passed to satisfy a signature but never called.

```java
// Java (01-ms-identity): the publisher is irrelevant for the reset-password scenario
new ResetPasswordUseCaseImpl(..., new DomainEventPublisher(null)); // dummy in this test path
```

### 2. Stub
Returns hardcoded responses. No call verification.

```python
# Python (06-ms-biometric): controls the scenario — always returns one template
class StubFacialRepository:
    async def find_by_person(self, person_id): return [TEMPLATE_FOR_TEST]
```

### 3. Fake
Real but simplified implementation. Has state, works correctly but lightweight.
**This is the dominant double in FaceAttend EDU.**

```java
// Real pattern from ResetPasswordUseCaseImplTest (01-ms-identity)
class FakeRecoveryCodes implements RecoveryCodePort {
    boolean verified;      // scenario control (stub-like knob)
    int consumeCalls;      // state observable by the test (spy-like bonus)

    @Override public void consume(String email) { consumeCalls++; }
    @Override public boolean isVerified(String email) { return verified; }
}
```

```python
# Real pattern from tests/support/in_memory_repositories.py (06-ms-biometric)
class InMemoryBiometricRepository:
    def __init__(self, kind: BiometricType): self.store: dict[str, list[Template]] = {}
    async def save(self, person_id, template): self.store.setdefault(person_id, []).append(template)
    async def find(self, person_id): return self.store.get(person_id, [])
```

### 4. Spy
Records the calls it receives. You can verify if it was called and with what arguments.

```typescript
// Real pattern from infrastructure/email/__tests__/sender.test.ts (08-ms-notification)
const sendMail = jest.fn().mockResolvedValue({});   // ← spy: records every call and its arguments
nodemailer.createTransport.mockReturnValue({ sendMail });

await sendEmail(cfg, { to: 'x@y.com', subject: 's', textBody: 't', htmlBody: '<p>h</p>' });

expect(sendMail).toHaveBeenCalledWith(expect.objectContaining({ to: 'x@y.com', subject: 's' }));
```

### 5. Mock
Has pre-programmed expectations. Fails if not called as expected.

```typescript
// Jest mock (for 03/07/09 where jest-mock is available)
const mockRepo = {
  save: jest.fn().mockResolvedValue(undefined),
  findById: jest.fn().mockResolvedValue(null),
};
expect(mockRepo.save).toHaveBeenCalledTimes(1);
```

### When to use each one?

| Double | When to use it | Used in repo by |
|--------|---------------|-----------------|
| Dummy | The collaborator does not matter in this test | identity use-case tests |
| Stub | You control the input scenario (what a collaborator returns) | biometric dependency overrides |
| Fake | Lightweight integration tests — behavior matters | ✅ everywhere (`Fake*`, `InMemory*`) |
| Spy | You verify that something was called (output effect) | ✅ `jest.fn()` sendMail spy (`sender.test.ts`) |
| Mock | You verify both input and output behavior | sparingly (jest only) |

**Preference:** Fake > Stub/Spy > Mock. Mocks are fragile — they break if you refactor the internal implementation.
The repo already follows this: `assertThat(recoveryCodes.consumeCalls).isEqualTo(1)` verifies effects on a Fake's state, not interactions on a mock framework.

---

## TDD by layer (with Hexagonal Architecture)

Hexagonal architecture makes TDD cheap here: the domain has zero framework imports, and
every outer concern enters through a port (`application/port/out`). If you cannot instantiate
a use case without a database, the architecture is wrong — fix the seam, not the test.

### Layer 1: Domain — Unit tests for aggregates

These are the most valuable tests. They test pure business logic.
**No infrastructure mocks.** No database. No HTTP.

```java
// REAL test in the repo: 01-ms-identity .../domain/service/UserActivationServiceTest.java
class UserActivationServiceTest {

    @Test
    void shouldActivateUserWhenPersonIsActive() {
        Person person = new Person();
        person.setStatus(true);
        User user = new User();
        user.setPersonId(person);

        new UserActivationService().activateUser(user);

        assertTrue(user.isActive());
    }

    @Test
    void shouldThrowExceptionWhenPersonIsInactive() {   // invariant: inactive person ⇒ no activation
        Person person = new Person();
        person.setStatus(false);
        User user = new User();
        user.setPersonId(person);

        assertThrows(DomainException.class, () -> new UserActivationService().activateUser(user));
    }
}
```

Domain rules worth pinning per service (candidates for new RED tests):
- **01-identity:** password policy validation, recovery-code expiry, session state machine.
- **05-attendance:** duplicate mark prevention, justification requires pending absence.
- **06-biometric:** cosine-distance match threshold, template versioning (`faceNet-v1`).

**TDD step:**
1. 🔴 Write `shouldThrowExceptionWhenPersonIsInactive` — fails because the rule is not implemented
2. 🟢 Implement `activateUser` with the minimum validation
3. ♻️ Refactor: extract exception message, rename for clarity

---

### Layer 2: Application — Use case tests

Test orchestration. Use Fakes for repositories and Spies for publishers.

```java
// REAL test in the repo: 01-ms-identity .../application/usecase/ResetPasswordUseCaseImplTest.java
@BeforeEach
void setUp() {
    userLookup = new FakeUserLookup();
    recoveryCodes = new FakeRecoveryCodes();
    userUpdate = new FakeUserUpdate();
    sessionUpdate = new FakeSessionUpdate();
    sessions = new FakeSessions();
    useCase = new ResetPasswordUseCaseImpl(
            userLookup, recoveryCodes, userUpdate, new FakePasswordHash(),
            new FakePolicies(), sessions, sessionUpdate, new DomainEventPublisher(null));
}

@Test
void resetsPasswordClosesSessionsAndConsumesTheChallenge() {
    User user = activeUser();
    userLookup.user = user;
    recoveryCodes.verified = true;                 // stub knob: scenario
    UserSession open = openSession(user);
    sessions.sessions = List.of(open);

    useCase.resetPassword(EMAIL, "Secret1!");

    assertThat(userUpdate.updated.getPasswordHash()).isEqualTo("hash:Secret1!"); // effect (fake state)
    assertThat(recoveryCodes.consumeCalls).isEqualTo(1);                          // spy-style count
    assertThat(open.getSessionStatus()).isEqualTo(UserSession.STATUS_CLOSED);     // side effect
}

@Test
void rejectsWhenThereIsNoVerifiedChallenge() {
    userLookup.user = activeUser();
    recoveryCodes.verified = false;

    assertThatThrownBy(() -> useCase.resetPassword(EMAIL, "Secret1!"))
            .isInstanceOf(UnauthorizedException.class);
    assertThat(userUpdate.updated).isNull();       // negative case: nothing was persisted
}
```

Note the AAA structure (`// Arrange / Act / Assert` also used in `academic.test.ts` and
`test_facial_embedding.py`) — it is the team convention for every new test.

---

### Layer 3: Infrastructure — Integration tests

Test that adapters interact correctly with external systems.
**Use the real database** (PostgreSQL schema of the service in Docker, migrations from `database/0X-ms-*-db` Liquibase).

```typescript
// tests/integration/SchoolRepositoryImpl.spec.ts (03-ms-academic — supertest+testcontainers deps installed)
describe('SchoolRepositoryImpl (integration with PostgreSQL)', () => {
  let db: DatabaseConnection;
  let repo: SchoolRepositoryImpl;

  beforeAll(async () => {
    db = await createTestDatabaseConnection(); // Testcontainers postgres:17
    await db.migrate();                        // same Liquibase changelog as prod
  });

  beforeEach(async () => {
    await db.query('TRUNCATE academic.schools CASCADE');
    repo = new SchoolRepositoryImpl(db);
  });

  it('saves and retrieves a school correctly', async () => {
    const created = await repo.save(SchoolFixture.createValid());
    const found = await repo.findById(created.schoolId);
    expect(found?.code).toBe(created.code);
  });

  it('soft-deletes instead of removing the row', async () => {
    const created = await repo.save(SchoolFixture.createValid());
    await repo.softDelete(created.schoolId);
    expect(await repo.findById(created.schoolId)).toBeNull();
    const raw = await db.query('SELECT deleted_at FROM academic.schools WHERE school_id=$1', [created.schoolId]);
    expect(raw.rows[0].deleted_at).not.toBeNull();
  });
});
```

Fallback when CI has no Docker: H2/in-memory profile (risk mitigation already written in `IEEE829_TEST_PLAN.md`).

---

### Layer 4: API — Contract tests (Consumer-Driven Contract Testing)

Verify that the OpenAPI contract is fulfilled in the real implementation.

```python
# REAL test in the repo: 06-ms-biometric/tests/unit/web/test_router_contract.py
EXPECTED_ROUTES = {
    ("POST", "/api/v1/biometric/facial/enroll"),
    ("GET",  "/api/v1/biometric/facial/{person_id}"),
    ("POST", "/api/v1/biometric/facial/verify"),
    ("PATCH","/api/v1/biometric/update-request/{request_id}/review"),
    ("GET",  "/health/ready"),
    # ... full pinned route table (21 endpoints)
}

@pytest.fixture
def client(stores):
    app.dependency_overrides[get_facial_repository] = lambda: stores["facial"]  # in-memory fakes
    # BIOMETRIC_MONGO_ENABLED=false → lifespan never touches the network
    with TestClient(app) as c:
        yield c

def test_enroll_returns_201_with_the_documented_shape(client):
    res = client.post("/api/v1/biometric/facial/enroll",
                      json={"person_id": PERSON, "encoding": UNIT_VECTOR})
    assert res.status_code == 201
    assert res.json()["template_version"] == "1.0"
```

Key idea already proven in the repo: **the router contract test pins the surface** —
route table, status codes (201/204), the ISO 9001 error envelope, and degraded-readiness
behavior — while running the *real* app with fakes behind the ports. Replicate this pattern
in TS with `app.inject()` + openapi-validator (see `testing-strategy.md`, Tier 3).

---

## Code coverage

Coverage is a signal, not an end in itself.

| Metric | Minimum target | Notes |
|--------|---------------|-------|
| Lines — domain | 90%+ | The business core must be well covered |
| Lines — application | 80%+ | Use cases must be covered |
| Lines — infrastructure | 60%+ | Integration tests cover the main paths |
| Branches | 75%+ | if/else must have tests for both paths |
| Global floor | ≥ 70% | IEEE 829 §1.4 acceptance criterion; > 80% on critical paths (ISTQB exit criteria) |

**What coverage does NOT tell you:**
- It does not say whether tests are meaningful
- It does not say whether you cover the right edge cases
- A test that only executes lines without assertions can give 100% but tests nothing

(Current gap: JaCoCo/coverage gates are not yet wired in CI — see `testing-strategy.md`.
Until then, reviewers enforce coverage qualitatively via the PR checklist.)

---

## TDD with Value Objects

VOs are the easiest objects to test with TDD. Start with them.

```java
// Target example mirroring 01-ms-identity DTO/domain style
class EmailTest {
    @Test void createsAValidEmail()        { assertDoesNotThrow(() -> Email.of("user@example.com")); }
    @Test void rejectsEmailWithoutAt()     { assertThrows(ValidationException.class, () -> Email.of("notvalid")); }
    @Test void normalizesToLowercase()     { assertEquals("user@example.com", Email.of("USER@EXAMPLE.COM").value()); }
    @Test void twoEqualEmailsAreEqual()    { assertEquals(Email.of("a@b.co"), Email.of("A@B.CO")); }
}
```

Existing VO-style coverage in the repo: `AuthRequestTest` (JSR-380 constraints on login DTO),
`BiometricType` enum behavior exercised through `test_router_contract.py` fixtures.

---

## TDD implementation order in a sprint

1. **Write domain tests first** (Aggregates, VOs, business rules)
2. **Implement the domain** until the tests pass
3. **Write use case tests** with Fakes for the ports (`application/port/out` interfaces exist from step 2's design)
4. **Implement the use case**
5. **Write integration tests** for the adapters
6. **Implement the adapters** (repository, publisher, HTTP controller)
7. **Write the contract test** for the HTTP endpoint (pin routes/statuses/error envelope)
8. **Refactor** at any point while the tests are green

Register each test with its IEEE 829 ID (`TC-05-012: …`) so traceability stays intact
(requirement ↔ case ↔ defect via `projectId`, per ISTQB report).

---

## Testing stack configuration

```json
// package.json (03/07/09 — Fastify services)
{
  "scripts": {
    "test": "vitest run",
    "test:unit": "vitest run src/__tests__",
    "test:integration": "vitest run tests/integration --no-file-parallelism",
    "test:coverage": "vitest run --coverage",
    "test:watch": "vitest"
  },
  "coverage": {
    "thresholds": { "lines": 80, "branches": 75 },
    "./src/domain/": { "lines": 90 }
  }
}
```

Other stacks: Maven Surefire runs JUnit 5 automatically on `mvn test` (add JaCoCo plugin for
gates); `pyproject.toml` configures `pytest` (`asyncio_mode = auto`, markers `unit`/`contract`);

---

## Correlations

- Testing pyramid & thresholds by tier → `11-quality/testing-strategy.md`
- Definition of Done (required coverage) → `00-governance/definition-of-done.md`
- Hexagonal Architecture (facilitates TDD) → `05-architecture/hexagonal-architecture.md`
- Code review checklist (tests as merge gate) → `11-quality/code-review.md`
- IEEE 829 plans/cases/incidents → `back-end/IEEE829_*.md`