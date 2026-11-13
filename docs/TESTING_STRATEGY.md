# TaskFlow API — Testing Strategy Document

**Document Version:** 1.2  
**Status:** Approved  
**Testing Stack:** JUnit 5, Mockito, AssertJ, Spring Boot Test, MockMvc, H2 (test profile) / Testcontainers (PostgreSQL 18-alpine)  

---

## 1. Testing Philosophy & Test Pyramid

The TaskFlow API testing strategy balances **execution speed, isolation, and environmental fidelity**.

```
          / \
         /   \      E2E Integration Tests (Full Spring Context + PostgreSQL)
        /     \     ---------------------------------------------------------
       / Web   \    Web Slice Tests (@WebMvcTest + MockMvc)
      / Slice   \   Data Slice Tests (@DataJpaTest + Testcontainers PostgreSQL)
     /           \  ---------------------------------------------------------
    /    Unit     \ Unit Tests (JUnit 5 + Mockito - Fast & Isolated)
   /_______________ \
```

### 1.1 Core Principles
1. **No Untested Migrations:** Flyway migrations must be executed during integration and repository test suites against a real PostgreSQL container.
2. **Behavioral Testing over Coverage Percentages:** Tests verify business requirements, edge cases, and constraint violations rather than aiming for blind line-coverage metrics.
3. **Deterministic & Isolated:** Tests must never depend on the outcome or execution order of other tests. Each integration test runs with clean database state.

---

## 2. Testing Levels & Tooling Matrix

| Level | Component Tested | Annotations / Tools | Isolation Strategy | Primary Verification |
| :--- | :--- | :--- | :--- | :--- |
| **Unit** | `TaskService`, `TaskMapper` | JUnit 5, AssertJ | Pure Java, in-memory stubs | Business rules, default assignments, entity transformations |
| **Filter** | `RequestCorrelationFilter` | JUnit 5, MockHttpServletRequest | Pure Java | `X-Request-ID` extraction, UUID generation, and MDC cleanup |
| **Web Slice** | `TaskController`, `HealthController`, `GlobalExceptionHandler` | MockMvc, standalone / `@WebMvcTest` | Standalone MockMvc | HTTP status codes, JSON serialization, ProblemDetail (400, 404, 503) |
| **Data Slice** | `TaskRepository` | Spring Data JPA | H2 / Testcontainers PostgreSQL | SQL queries, status ordering, DB constraints |
| **Integration** | Full Request Lifecycle | `@SpringBootTest`, MockMvc | Real Spring Context | Complete end-to-end flows (HTTP -> Service -> DB -> HTTP) |

---

## 3. Database Testing Strategy: Real PostgreSQL via Testcontainers

### 3.1 Why Testcontainers over H2?
While H2 in-memory is fast, it differs from PostgreSQL in:
- SQL syntax compatibility (e.g., PostgreSQL `ILIKE`, `TIMESTAMPTZ`, identity columns).
- Flyway migration execution (disabling Flyway leaves production migration scripts untested).
- Case-insensitivity rules and index utilization.

### 3.2 Test Configuration (`src/test/resources/application-test.yml`)

Using Spring Boot's automatic Testcontainers support:

```yaml
spring:
  datasource:
    url: jdbc:tc:postgresql:16-alpine:///taskflow_test
    driver-class-name: org.testcontainers.jdbc.ContainerDatabaseDriver
  jpa:
    hibernate:
      ddl-auto: validate
    properties:
      hibernate:
        dialect: org.hibernate.dialect.PostgreSQLDialect
  flyway:
    enabled: true
    locations: classpath:db/migration
```

---

## 4. Test Case Catalog by Component

### 4.1 Unit Tests (`TaskServiceTest`)
*Dependencies mocked via `@Mock private TaskRepository taskRepository;`*

| Scenario | Input | Expected Outcome |
| :--- | :--- | :--- |
| Create valid task | `CreateTaskRequest("Learn Spring", ...)` | Entity persisted with `status=PENDING`, returns `TaskResponse` |
| Create task with omitted priority | `CreateTaskRequest(priority=null)` | Persisted entity assigned default `priority=MEDIUM` |
| Retrieve existing task | `id = 1L` (mock returns entity) | Returns mapped `TaskResponse` |
| Retrieve missing task | `id = 999L` (mock returns empty) | Throws `TaskNotFoundException` |
| Update existing task | `id = 1L`, `UpdateTaskRequest(...)` | Entity fields modified, `updatedAt` refreshed, returns `TaskResponse` |
| Update missing task | `id = 999L` (mock returns empty) | Throws `TaskNotFoundException` |
| Complete pending task | `id = 1L` (status `PENDING`) | Status transitions to `COMPLETED`, returns `TaskResponse` |
| Complete already-completed task | `id = 1L` (status `COMPLETED`) | Idempotent: returns `TaskResponse` with status `COMPLETED` |
| Delete existing task | `id = 1L` | Invokes `repository.delete(task)` |
| Delete missing task | `id = 999L` | Throws `TaskNotFoundException` |

### 4.2 Controller Tests (`TaskControllerTest` & `HealthControllerTest`)
*Tested via `MockMvc`.*

| HTTP Request | Condition / Mock Setup | Expected HTTP Status | Expected JSON Structure |
| :--- | :--- | :--- | :--- |
| `POST /api/v1/tasks` | Valid payload | `201 Created` | Header `Location`, body with `id`, `title`, `status` |
| `POST /api/v1/tasks` | Blank title (`""`) | `400 Bad Request` | ProblemDetail with `invalidParams.title` |
| `POST /api/v1/tasks` | Title > 150 characters | `400 Bad Request` | ProblemDetail with `invalidParams.title` |
| `GET /api/v1/tasks/1` | Service returns `TaskResponse` | `200 OK` | Task JSON object |
| `GET /api/v1/tasks/999` | Service throws `TaskNotFoundException` | `404 Not Found` | ProblemDetail with status 404 |
| `GET /api/v1/tasks` | Default list retrieval | `200 OK` | Array of Task JSON objects |
| `PUT /api/v1/tasks/1` | Valid update payload | `200 OK` | Updated Task JSON object |
| `DELETE /api/v1/tasks/1` | Service completes successfully | `204 No Content` | Empty response body |
| `PATCH /api/v1/tasks/1/complete` | Service returns completed task | `200 OK` | Task JSON object with `status: "COMPLETED"` |
| `GET /api/v1/health` | Database connection valid | `200 OK` | `status: "UP"`, `details.database: "UP"` |
| `GET /api/v1/health` | Database connection failing | `503 Service Unavailable` | `status: "DOWN"`, `details.database: "DOWN"` |

### 4.3 Repository & Query Tests (`TaskRepositoryTest`)
*Tested against real PostgreSQL via Testcontainers.*

| Query Tested | Seed Data | Assertion |
| :--- | :--- | :--- |
| Save and retrieve by ID | Single task entity | Retrieved entity matches persisted fields and timestamps |
| Filter by `status = PENDING` | 3 `PENDING`, 2 `COMPLETED` | Returns exactly 3 records |
| Filter by `priority = HIGH` | 1 `HIGH`, 2 `MEDIUM` | Returns exactly 1 record |
| Filter by `dueDate = 2026-11-20` | Matching and non-matching dates | Returns only records on target date |
| Title search `title = "Spring"` | `"Learn Spring Boot"`, `"Read Docs"` | Case-insensitive substring match returns `"Learn Spring Boot"` |
| Combined filter | Status `PENDING` + Priority `HIGH` | Returns intersection of both predicates |
| Pagination & Sorting | 15 tasks, request `page=0&size=10` sorted `createdAt,desc` | Returns first 10 tasks in descending order with correct count |

---

## 5. Test Execution Commands

```bash
# Execute entire test suite
./mvnw clean test

# Execute single test class
./mvnw -Dtest=TaskServiceTest test

# Execute controller tests only
./mvnw -Dtest=TaskControllerTest test

# Execute repository integration tests only
./mvnw -Dtest=TaskRepositoryTest test
```

---

## 6. Coverage & Quality Gates

Before merging into `main`:
1. **Zero Test Failures:** 100% of tests must pass.
2. **Flyway Verification:** Flyway must migrate successfully against PostgreSQL in tests.
3. **Behavioral Coverage Targets:**
   - Service business logic: 100% coverage of conditions and branches.
   - Validation & Controller mapping: 100% coverage of `@Valid` rules and error responses.
   - Dynamic query predicates: All 4 filter parameters tested individually and in combination.
