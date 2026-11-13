# TaskFlow API — Product Requirements Document (PRD)

**Document Version:** 1.3  
**Status:** Approved  
**Project Type:** Backend REST API  
**Product Name:** TaskFlow API  
**Target Runtime:** Java 21 LTS, Spring Boot 4.1.2, PostgreSQL 18-alpine  

---

## 1. Product Overview

TaskFlow API is a lightweight, high-performance task management backend service. It provides robust RESTful endpoints enabling users and consuming client applications to create, retrieve, update, delete, complete, search, filter, and paginate tasks.

All data is reliably persisted in a PostgreSQL relational database with strong schema-level constraints and database migrations managed via Flyway.

---

## 2. Problem Statement

Individual developers and client applications require a reliable, standard-compliant backend API to track and manage tasks without the unnecessary overhead of complex third-party tools.

Core capabilities required:
- Capture tasks with explicit titles, optional descriptions, priorities, and deadlines.
- Track task lifecycle states (`PENDING` -> `COMPLETED`).
- Fast search and multi-criteria filtering (by status, priority, due date, and title).
- Scalable paginated access with flexible sorting to handle growing task volumes.
- Clean RESTful design with standard status codes and structured validation errors.

---

## 3. Product Goals

1. **Standard REST API:** Deliver an intuitive, resource-oriented REST interface adhering to standard HTTP verbs and status codes.
2. **Data Integrity:** Enforce data validation at both the API boundary (Jakarta Validation) and persistence tier (PostgreSQL constraints).
3. **Predictable Performance:** Support database-level pagination, sorting, and indexing for low-latency queries on common filter paths.
4. **Maintainable Architecture:** Implement a layered architecture organized by feature (package-by-feature) with clean separation between DTOs and domain entities.
5. **Automated Verification:** Maintain automated test suites covering unit, web layer, and repository integration levels.

---

## 4. Target Audience & Scope Boundaries

### 4.1 Target Audience
- Frontend developers building web or mobile task management clients.
- Backend engineers integrating task tracking into automated workflows or developer tooling.

### 4.2 In-Scope Features (v1)
- Check system health and database connectivity (`GET /api/v1/health`).
- Create a task with title, optional description, priority, and optional due date.
- Retrieve a single task by its unique identifier.
- List tasks with database-level pagination and sorting.
- Update editable attributes of an existing task (`title`, `description`, `priority`, `dueDate`, `status`).
- Permanently delete a task.
- Mark a task as completed via an idempotent action endpoint.
- Filter tasks by status (`PENDING`, `COMPLETED`).
- Filter tasks by priority (`LOW`, `MEDIUM`, `HIGH`).
- Filter tasks by due date (`YYYY-MM-DD`).
- Case-insensitive search on task titles.
- Combine multiple filters concurrently.
- Consistent error payloads with field-level validation breakdowns.

### 4.3 Explicitly Excluded Features (Out of Scope for v1)
The following capabilities are deliberately excluded to maintain a sharp, focused scope:
- User registration, login, and JWT/OAuth2 authentication.
- Multi-tenancy and user-specific task ownership (all clients share the single dataset in v1).
- Soft deletion / Trash bin recovery.
- Task categories, tags, or file attachments.
- Subtasks and dependency trees.
- Asynchronous notifications, email alerts, or webhooks.
- Redis caching, Kafka/RabbitMQ message brokers.
- Microservice decomposition or distributed tracing.

---

## 5. User Stories & Acceptance Criteria

### US-01: Create a Task
**As a** client application,  
**I want to** create a new task with a title, description, priority, and due date,  
**So that** I can track pending work.

**Acceptance Criteria:**
- The request payload must include a non-blank `title` (max 150 characters).
- `description` is optional (supports large text).
- `priority` is optional; if omitted, defaults to `MEDIUM`. Valid values: `LOW`, `MEDIUM`, `HIGH`.
- `dueDate` is optional; if present, must follow `YYYY-MM-DD`.
- Initial `status` is automatically assigned to `PENDING`.
- Auto-generates `id`, `createdAt`, and `updatedAt` timestamps in UTC.
- Returns HTTP `201 Created` with a `Location` header and the created task response body.

### US-02: Retrieve a Task by ID
**As a** client application,  
**I want to** fetch a task by its numeric ID,  
**So that** I can inspect its complete details.

**Acceptance Criteria:**
- Returns HTTP `200 OK` with the task DTO if found.
- Returns HTTP `404 Not Found` with a standard error body if the ID does not exist.
- Rejects non-numeric IDs with HTTP `400 Bad Request`.

### US-03: Retrieve Paginated Tasks
**As a** client application,  
**I want to** view a page of tasks with pagination metadata,  
**So that** I can present data efficiently without loading the entire database.

**Acceptance Criteria:**
- Supports query parameters: `page` (default: 0), `size` (default: 10, max: 100), `sort` (default: `createdAt,desc`).
- Returns HTTP `200 OK` with a standardized page response containing `content` items and metadata (`page`, `size`, `totalElements`, `totalPages`, `first`, `last`).
- Never loads entire tables into memory.

### US-04: Update an Existing Task
**As a** client application,  
**I want to** update an existing task's attributes,  
**So that** I can reflect revisions or progress.

**Acceptance Criteria:**
- Accepts `PUT /api/v1/tasks/{id}` with complete editable attributes.
- Validates all fields against domain constraints (non-blank title, valid enum values).
- Updates `updatedAt` to the current UTC timestamp.
- Returns HTTP `200 OK` with the updated task.
- Returns HTTP `404 Not Found` if the task does not exist.

### US-05: Delete a Task
**As a** client application,  
**I want to** permanently remove a task,  
**So that** obsolete records do not clutter my view.

**Acceptance Criteria:**
- Accepts `DELETE /api/v1/tasks/{id}`.
- Permanently deletes the record from the database.
- Returns HTTP `204 No Content` with an empty response body on success.
- Returns HTTP `404 Not Found` if the task does not exist.

### US-06: Mark a Task as Complete
**As a** client application,  
**I want to** trigger a completion action on a task,  
**So that** I can quickly record that work is finished without sending the full payload.

**Acceptance Criteria:**
- Accepts `PATCH /api/v1/tasks/{id}/complete`.
- Sets `status` to `COMPLETED` and refreshes `updatedAt`.
- **Idempotency:** Calling the endpoint on an already completed task succeeds and returns HTTP `200 OK` with status `COMPLETED`.
- Returns HTTP `404 Not Found` if the task does not exist.

### US-07: Filter and Search Tasks
**As a** client application,  
**I want to** filter tasks by status, priority, due date, and search title keywords,  
**So that** I can locate relevant tasks immediately.

**Acceptance Criteria:**
- Filter by status (`?status=PENDING` or `?status=COMPLETED`).
- Filter by priority (`?priority=LOW`, `MEDIUM`, or `HIGH`).
- Filter by due date (`?dueDate=YYYY-MM-DD`).
- Search by title (`?title=spring`): case-insensitive substring match.
- Supports combining any or all filters simultaneously with pagination and sorting.

---

## 6. Task Domain Specification

### 6.1 Task Attributes
| Field Name | Storage Type | Nullable | Default | Description |
| :--- | :--- | :--- | :--- | :--- |
| `id` | `BIGINT` | No | Auto Identity | Unique task identifier |
| `title` | `VARCHAR(150)` | No | None | Concise task summary (1-150 chars) |
| `description` | `TEXT` | Yes | `NULL` | Detailed notes or specifications |
| `status` | `VARCHAR(20)` | No | `'PENDING'` | Lifecycle status (`PENDING`, `COMPLETED`) |
| `priority` | `VARCHAR(20)` | No | `'MEDIUM'` | Urgency level (`LOW`, `MEDIUM`, `HIGH`) |
| `due_date` | `DATE` | Yes | `NULL` | Target completion date (`YYYY-MM-DD`) |
| `created_at` | `TIMESTAMPTZ` | No | `CURRENT_TIMESTAMP` | UTC creation timestamp |
| `updated_at` | `TIMESTAMPTZ` | No | `CURRENT_TIMESTAMP` | UTC last modification timestamp |

---

## 7. Non-Functional Requirements

### 7.1 Maintainability & Architecture
- Layered architecture: `Controller -> Service -> Repository -> Database`.
- Organization: Package-by-feature (`com.taskflow.task.*`).
- Complete isolation between API DTOs and internal JPA Entities.
- Strict constructor-based dependency injection.

### 7.2 Reliability & Data Consistency
- Service operations mutating data must execute within a `@Transactional` boundary.
- Database enforces check constraints on status and priority enums.

### 7.3 Performance & Indexing
- Pagination enforced at the SQL layer via `LIMIT` and `OFFSET`.
- Composite indexing aligned with default sorting and filter access paths.

### 7.4 Quality & Test Coverage
- Unit tests with JUnit 5 and Mockito for business rules.
- HTTP integration tests with MockMvc for web controllers.
- Native PostgreSQL repository tests using Testcontainers.

---

## 8. Success Criteria

1. All functional requirements (US-01 through US-07) implemented and verified.
2. Flyway migration scripts cleanly build the schema from scratch.
3. Automated test suite passes with zero failures.
4. OpenAPI 3.0 / Swagger documentation accessible at `/swagger-ui.html`.
5. Docker Compose environment launches PostgreSQL and runs the application seamlessly.
