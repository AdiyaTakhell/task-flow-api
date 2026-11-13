# TaskFlow API — Software Requirements Specification (SRS)

**Document Version:** 1.2  
**Status:** Approved  
**Related Documents:** [PRD.md](PRD.md), [DATABASE_DESIGN.md](DATABASE_DESIGN.md), [API_DESIGN.md](API_DESIGN.md), [ARCHITECTURE.md](ARCHITECTURE.md)  
**Project Type:** Backend REST API  
**Target Runtime:** Java 21 LTS, Spring Boot 4.1.2, PostgreSQL 18-alpine  

---

## 1. Introduction

### 1.1 Purpose
This document specifies the software requirements for the TaskFlow REST API. It establishes the authoritative technical contract for software engineers, QA specialists, and architects building, reviewing, and testing the system.

### 1.2 System Scope
TaskFlow API is a standalone backend application that exposes resource-oriented HTTP endpoints for managing tasks. The service handles input validation, coordinates business workflows, enforces database transactions, and manages persistent state in PostgreSQL.

Version 1 operates with a shared single-tenant dataset without user authentication or multi-user access controls.

### 1.3 Definitions and Standards
- **REST:** Representational State Transfer
- **DTO:** Data Transfer Object (separates API surface from JPA persistence entities)
- **UTC:** Coordinated Universal Time
- **ISO-8601:** Standard representation for dates and timestamps (`YYYY-MM-DD` and `YYYY-MM-DD'T'HH:mm:ssX`)
- **Jakarta Validation:** Standardized declarative bean validation (`@NotNull`, `@NotBlank`, `@Size`)

---

## 2. Overall System Description

### 2.1 Architectural Pattern
The application follows a **Layered Architecture organized by Feature**:
1. **Controller Layer:** Translates HTTP requests, handles serialization/deserialization, and invokes bean validation.
2. **Service Layer:** Houses all business rules, orchestration logic, and transactional boundaries.
3. **Repository Layer:** Abstract data access layer utilizing Spring Data JPA and dynamic criteria predicates.
4. **Database Tier:** PostgreSQL 16+ relational storage managed by Flyway versioned migrations.

### 2.2 Operating Environment
- **JDK:** Java 21 LTS
- **Framework:** Spring Boot 4.1.2
- **RDBMS:** PostgreSQL 18-alpine
- **Build Tool:** Apache Maven 3.9+ (via `mvnw`)
- **Migration Engine:** Flyway 10+
- **Container Engine:** Docker & Docker Compose

---

## 3. Functional Requirements

### 3.1 Task Creation
* **Requirement ID:** `SRS-FR-001`
* **HTTP Method & Path:** `POST /api/v1/tasks`
* **Input Parameters:**
  - `title` (String, required, 1-150 characters, non-blank)
  - `description` (String, optional, max 2000 characters)
  - `priority` (Enum: `LOW`, `MEDIUM`, `HIGH`; optional, defaults to `MEDIUM`)
  - `dueDate` (Date: `YYYY-MM-DD`, optional, may be past or future)
* **System Actions:**
  1. Validates request body; returns `400 Bad Request` with field details on failure.
  2. Assigns default `status = 'PENDING'` if not specified.
  3. Assigns default `priority = 'MEDIUM'` if omitted.
  4. Generates an auto-incrementing `BIGINT` ID.
  5. Sets `createdAt` and `updatedAt` to current UTC timestamp.
  6. Persists record to the `tasks` table.
* **Response:** HTTP `201 Created` with `Location: /api/v1/tasks/{id}` header and created `TaskResponse` body.

### 3.2 Task Retrieval by Identifier
* **Requirement ID:** `SRS-FR-002`
* **HTTP Method & Path:** `GET /api/v1/tasks/{id}`
* **System Actions:**
  1. Extracts `{id}` from path.
  2. Queries `tasks` table.
  3. Returns HTTP `200 OK` with `TaskResponse` if found.
  4. Returns HTTP `404 Not Found` if record does not exist.
  5. Returns HTTP `400 Bad Request` if `{id}` is not a valid positive 64-bit integer.

### 3.3 List Tasks (Filtering, Searching, Pagination, Sorting)
* **Requirement ID:** `SRS-FR-003`
* **HTTP Method & Path:** `GET /api/v1/tasks`
* **Supported Query Parameters:**
  | Parameter | Type | Required | Default | Constraint / Description |
  | :--- | :--- | :--- | :--- | :--- |
  | `page` | Integer | No | `0` | $\ge 0$ |
  | `size` | Integer | No | `10` | $1 \le \text{size} \le 100$ |
  | `sort` | String | No | `createdAt,desc` | Format: `<field>,<asc\|desc>`. Allowed fields: `id`, `title`, `status`, `priority`, `dueDate`, `createdAt`, `updatedAt` |
  | `status` | String | No | None | Enum: `PENDING`, `COMPLETED` |
  | `priority` | String | No | None | Enum: `LOW`, `MEDIUM`, `HIGH` |
  | `dueDate` | Date | No | None | Format: `YYYY-MM-DD` |
  | `title` | String | No | None | Case-insensitive substring match |
* **System Actions:**
  1. Validates query parameters; returns `400 Bad Request` if unsupported sort field or invalid enum is provided.
  2. Dynamically composes SQL `WHERE` predicates using Spring Data JPA Specification.
  3. Executes a paginated SQL query using database `LIMIT` and `OFFSET`.
  4. Returns HTTP `200 OK` with a standardized `TaskPageResponse` payload containing metadata.

### 3.4 Update Task (Full Replacement)
* **Requirement ID:** `SRS-FR-004`
* **HTTP Method & Path:** `PUT /api/v1/tasks/{id}`
* **Input Parameters:**
  - `title` (String, required, 1-150 characters, non-blank)
  - `description` (String, optional, max 2000 characters)
  - `status` (Enum: `PENDING`, `COMPLETED`, required)
  - `priority` (Enum: `LOW`, `MEDIUM`, `HIGH`, required)
  - `dueDate` (Date: `YYYY-MM-DD`, optional)
* **System Actions:**
  1. Validates request body and verifies `{id}` exists in the database.
  2. Returns `404 Not Found` if ID does not exist.
  3. Overwrites existing entity attributes with new values.
  4. Updates `updatedAt` timestamp to current UTC time.
  5. Commits transaction and returns HTTP `200 OK` with updated `TaskResponse`.

### 3.5 Delete Task
* **Requirement ID:** `SRS-FR-005`
* **HTTP Method & Path:** `DELETE /api/v1/tasks/{id}`
* **System Actions:**
  1. Verifies task exists; returns `404 Not Found` if not found.
  2. Executes hard deletion (`DELETE FROM tasks WHERE id = ?`).
  3. Returns HTTP `204 No Content` with an empty response body.

### 3.6 Complete Task Action
* **Requirement ID:** `SRS-FR-006`
* **HTTP Method & Path:** `PATCH /api/v1/tasks/{id}/complete`
* **System Actions:**
  1. Verifies task exists; returns `404 Not Found` if missing.
  2. Transitions `status` to `COMPLETED`.
  3. If already `COMPLETED`, performs no state change (idempotent behavior).
  4. Updates `updatedAt` timestamp.
  5. Returns HTTP `200 OK` with the updated `TaskResponse`.

### 3.7 Health Check
* **Requirement ID:** `SRS-FR-007`
* **HTTP Method & Path:** `GET /api/v1/health`
* **System Actions:**
  1. Validates application uptime and probes database connection via `DataSource.getConnection().isValid(timeout)`.
  2. Returns HTTP `200 OK` with `status: "UP"` when database connectivity is healthy.
  3. Returns HTTP `503 Service Unavailable` with `status: "DOWN"` if database connection fails.

---

## 4. Data Storage & Schema Requirements

### 4.1 Table Definition (`tasks`)
```sql
CREATE TABLE tasks (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    title VARCHAR(150) NOT NULL,
    description TEXT,
    status VARCHAR(20) NOT NULL DEFAULT 'PENDING',
    priority VARCHAR(20) NOT NULL DEFAULT 'MEDIUM',
    due_date DATE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP,
    CONSTRAINT chk_tasks_title_not_blank CHECK (length(trim(title)) > 0),
    CONSTRAINT chk_tasks_status CHECK (status IN ('PENDING', 'COMPLETED')),
    CONSTRAINT chk_tasks_priority CHECK (priority IN ('LOW', 'MEDIUM', 'HIGH'))
);
```

### 4.2 Indexes
```sql
-- Optimal composite index for default listing and status filtering
CREATE INDEX idx_tasks_status_created ON tasks (status, created_at DESC);

-- Index for due date filtering and range checks
CREATE INDEX idx_tasks_due_date ON tasks (due_date);
```

### 4.3 Flyway & Hibernate Configuration
- Database migrations are strictly managed using Flyway versioned scripts (`V1__create_tasks_table.sql`).
- Hibernate DDL execution is locked to validation only in production and standard runs:
  ```properties
  spring.jpa.hibernate.ddl-auto=validate
  spring.flyway.enabled=true
  ```

---

## 5. Input Validation Rules

| Attribute | Constraint | Failure HTTP Code | Error Message |
| :--- | :--- | :--- | :--- |
| `title` | `@NotBlank`, `@Size(max = 150)` | `400 Bad Request` | "Title is required and must not exceed 150 characters" |
| `description` | `@Size(max = 2000)` | `400 Bad Request` | "Description cannot exceed 2000 characters" |
| `status` | Must match valid `TaskStatus` enum | `400 Bad Request` | "Invalid status. Allowed values: PENDING, COMPLETED" |
| `priority` | Must match valid `TaskPriority` enum | `400 Bad Request` | "Invalid priority. Allowed values: LOW, MEDIUM, HIGH" |
| `dueDate` | Valid ISO-8601 `LocalDate` (`YYYY-MM-DD`) | `400 Bad Request` | "Due date must be in YYYY-MM-DD format" |
| `page` | Minimum `0` | `400 Bad Request` | "Page index must not be less than zero" |
| `size` | Range: `1` to `100` | `400 Bad Request` | "Page size must be between 1 and 100" |
| `sort` | Approved property and direction | `400 Bad Request` | "Invalid sort property or direction" |

---

## 6. Error Handling Specifications

All error responses adhere to RFC 7807/9457 `ProblemDetail`:

```json
{
  "type": "urn:problem:validation-failed",
  "title": "Validation Failed",
  "status": 400,
  "detail": "Request validation failed on one or more fields",
  "timestamp": "2026-11-13T14:30:00Z",
  "invalidParams": {
    "title": "Title is required and cannot be blank"
  }
}
```

### Handled Exception Hierarchy:
1. `MethodArgumentNotValidException` -> HTTP 400 with populated `invalidParams` map.
2. `ConstraintViolationException` / `MethodArgumentTypeMismatchException` -> HTTP 400 with description.
3. `TaskNotFoundException` -> HTTP 404 with missing task identifier.
4. `HttpMessageNotReadableException` (malformed JSON / invalid enum values) -> HTTP 400.
5. `Exception` (catch-all unexpected internal errors) -> HTTP 500 with sanitized generic message (no stack traces leaked).

---

## 7. Requirement Traceability Matrix

| Business Requirement | SRS Requirement ID | Controller Method | Service Method | Verification Test |
| :--- | :--- | :--- | :--- | :--- |
| Create Task | `SRS-FR-001` | `createTask` | `createTask` | `TaskControllerTest.createTask_validRequest_returns201Created` |
| Get Task by ID | `SRS-FR-002` | `getTaskById` | `getTaskById` | `TaskControllerTest.getTaskById_exists_returns200` |
| List Tasks / Filters | `SRS-FR-003` | `getAllTasks` | `getTasks` | `TaskServiceTest.getTasks_withStatus_shouldFilterByStatus` |
| Update Task | `SRS-FR-004` | `updateTask` | `updateTask` | `TaskControllerTest.updateTask_validRequest_returns200` |
| Delete Task | `SRS-FR-005` | `deleteTask` | `deleteTask` | `TaskControllerTest.deleteTask_returns204NoContent` |
| Complete Task | `SRS-FR-006` | `completeTask` | `completeTask` | `TaskControllerTest.completeTask_returns200WithCompletedStatus` |
| Health Check | `SRS-FR-007` | `checkHealth` | `checkDatabase` | `HealthControllerTest.checkHealth_whenDatabaseHealthy_returns200AndStatusUp` |
