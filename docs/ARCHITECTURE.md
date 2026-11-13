# TaskFlow API — Application Architecture Document

**Document Version:** 1.2  
**Status:** Approved  
**Pattern:** Layered Architecture with Package-by-Feature Organization  
**Runtime:** Java 21 LTS, Spring Boot 4.1.2, PostgreSQL 18-alpine  

---

## 1. Architectural Principles

TaskFlow API is built around six core software engineering rules:
1. **Layered Separation of Concerns:** Strict boundaries between Presentation (Controller), Business Logic (Service), and Persistence (Repository).
2. **Package-by-Feature Organization:** Feature components are grouped under `com.taskflow.api.tasks.*`, while cross-cutting concerns reside in `com.taskflow.api.common.*`, `config.*`, and `health.*`.
3. **Encapsulated Domain Entities:** JPA Entities never leak to Controllers or HTTP clients. Immutable Java `record` DTOs form the API boundary.
4. **Service-Owned Transactions:** The Service layer owns transactions via `@Transactional(readOnly = true)` at class level and `@Transactional` on mutations.
5. **Standardized Problem Details:** Error responses adhere to RFC 7807/9457 `ProblemDetail` format via `@RestControllerAdvice`.
6. **Observability & Request Correlation:** Inbound HTTP requests receive unique correlation IDs (`X-Request-ID`) tracked through SLF4J MDC across all log outputs.

---

## 2. Package Structure

```
com.taskflow.api
│
├── TaskFlowApiApplication.java
│
├── config
│   └── OpenApiConfig.java
│
├── health
│   ├── controller
│   │   └── HealthController.java
│   └── dto
│       └── HealthResponse.java
│
├── tasks
│   ├── controller
│   │   └── TaskController.java
│   │
│   ├── service
│   │   └── TaskService.java
│   │
│   ├── repository
│   │   └── TaskRepository.java
│   │
│   ├── entity
│   │   └── Task.java
│   │
│   ├── dto
│   │   ├── CreateTaskRequest.java
│   │   ├── UpdateTaskRequest.java
│   │   └── TaskResponse.java
│   │
│   ├── mapper
│   │   └── TaskMapper.java
│   │
│   ├── enums
│   │   ├── TaskStatus.java
│   │   └── TaskPriority.java
│   │
│   └── exception
│       └── TaskNotFoundException.java
│
└── common
    ├── exception
    │   └── GlobalExceptionHandler.java
    │
    ├── response
    │   └── ApiResponse.java
    │
    └── filter
        └── RequestCorrelationFilter.java
```

---

## 3. Layer Responsibilities & Component Design

### 3.1 Presentation Layer (`TaskController`, `HealthController`)
- **Role:** HTTP transport adapter.
- **Responsibilities:**
  - Exposes REST endpoints (`@RestController`, `@RequestMapping`).
  - Triggers declarative input validation via `@Valid`.
  - Dispatches calls directly to domain services.
  - Returns DTO records (`TaskResponse`, `HealthResponse`) wrapped in `ResponseEntity` with standard status codes (`200`, `201`, `204`, `503`).
- **Rule:** Controllers never reference JPA entities directly.

### 3.2 Service Layer (`TaskService`)
- **Role:** Business logic execution and transaction management.
- **Responsibilities:**
  - Class-level `@Transactional(readOnly = true)` with write methods annotated with `@Transactional`.
  - Maps requests to entities and entities to responses via `TaskMapper`.
  - Performs existence checks and throws domain exceptions (`TaskNotFoundException`).
  - Employs SLF4J structured logging for lifecycle events and state transitions.

### 3.3 Mapper Component (`TaskMapper`)
- **Role:** High-performance, zero-reflection data mapping.
- **Responsibilities:**
  - Maps `CreateTaskRequest` -> `Task` entity.
  - Updates existing `Task` entities from `UpdateTaskRequest`.
  - Maps `Task` entity -> `TaskResponse` DTO record.
  - Maps collections `List<Task>` -> `List<TaskResponse>`.

### 3.4 Persistence Layer (`TaskRepository`, `Task`)
- **Role:** Relational data access and schema mapping.
- **Responsibilities:**
  - Extends Spring Data JPA `JpaRepository<Task, Long>`.
  - Provides queries such as `findAllByOrderByCreatedAtDesc()` and `findAllByStatusOrderByCreatedAtDesc(TaskStatus status)`.
  - Maps database table `tasks` with column constraints, pre-persist auditing, and composite index annotations matching Flyway migrations.

### 3.5 Cross-Cutting Infrastructure
- **Request Correlation (`RequestCorrelationFilter`):** Intercepts requests, reads or generates `X-Request-ID`, populates SLF4J MDC, exposes the header in the response, and cleans up MDC in a `finally` block.
- **Global Error Handling (`GlobalExceptionHandler`):** Translates exceptions into standard RFC 7807/9457 `ProblemDetail` structures (`404` for missing tasks, `400` with `invalidParams` map for Bean Validation errors, `500` for unexpected errors).
- **API Documentation (`OpenApiConfig`):** Configures Swagger UI and OpenAPI 3.0 metadata accessible at `/swagger-ui/index.html`.
