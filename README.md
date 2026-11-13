# TaskFlow API

TaskFlow API is a lightweight, high-performance task management REST API built with Java 21, Spring Boot 4.1.2, and PostgreSQL 18.

---

## Features

- **Resource-Oriented REST API:** Intuitive CRUD endpoints with standard HTTP methods and status codes.
- **Data Validation & RFC 7807/9457 Errors:** Declarative Bean Validation with standardized `ProblemDetail` responses.
- **PostgreSQL Persistence & Flyway:** Robust schema evolution with check constraints and index optimization.
- **Request Tracing & Correlation:** Inbound `X-Request-ID` propagation and SLF4J MDC logging.
- **Health Verification:** Native health endpoint at `/api/v1/health` verifying application and database connectivity.
- **OpenAPI 3 / Swagger Documentation:** Interactive API explorer accessible at `/swagger-ui/index.html`.

---

## Tech Stack

| Component | Technology | Version |
|---|---|---|
| Language | Java (LTS) | 21 |
| Framework | Spring Boot | 4.1.2 |
| Database | PostgreSQL | 18-alpine |
| Migration Engine | Flyway | 10.x |
| Persistence / ORM | Spring Data JPA / Hibernate | 7.x |
| Connection Pool | HikariCP | Default |
| API Documentation | Springdoc OpenAPI | 3.2.0 |
| Testing | JUnit 5, AssertJ, MockMvc | Modern Stack |

---

## Project Structure

The project follows a **Layered Architecture organized by Feature**:

```
src/main/java/com/taskflow/api
├── TaskFlowApiApplication.java
├── config
│   └── OpenApiConfig.java
├── health
│   ├── controller
│   │   └── HealthController.java
│   └── dto
│       └── HealthResponse.java
├── tasks
│   ├── controller
│   │   └── TaskController.java
│   ├── service
│   │   └── TaskService.java
│   ├── repository
│   │   └── TaskRepository.java
│   ├── entity
│   │   └── Task.java
│   ├── dto
│   │   ├── CreateTaskRequest.java
│   │   ├── UpdateTaskRequest.java
│   │   └── TaskResponse.java
│   ├── mapper
│   │   └── TaskMapper.java
│   ├── enums
│   │   ├── TaskStatus.java
│   │   └── TaskPriority.java
│   └── exception
│       └── TaskNotFoundException.java
└── common
    ├── exception
    │   └── GlobalExceptionHandler.java
    ├── response
    │   └── ApiResponse.java
    └── filter
        └── RequestCorrelationFilter.java
```

---

## Quick Start

### 1. Start PostgreSQL with Docker Compose
```bash
docker compose up -d
```

### 2. Build and Run the Application
```bash
# Windows
.\mvnw.cmd spring-boot:run

# Linux / macOS
./mvnw spring-boot:run
```

The application starts on `http://localhost:8080`.

### 3. Explore API Documentation
Open your browser and navigate to:
```
http://localhost:8080/swagger-ui/index.html
```

---

## API Endpoints

| HTTP Method | Endpoint | Description | Success Code |
|---|---|---|---|
| `GET` | `/api/v1/health` | Check application and database health | `200 OK` |
| `POST` | `/api/v1/tasks` | Create a new task (defaults to `PENDING`) | `201 Created` |
| `GET` | `/api/v1/tasks` | Paginated task list with sorting and filters (`page`, `size`, `sort`, `status`, `priority`, `dueDate`, `title`) | `200 OK` |
| `GET` | `/api/v1/tasks/{id}` | Retrieve a single task by ID | `200 OK` |
| `PUT` | `/api/v1/tasks/{id}` | Update task details | `200 OK` |
| `PATCH` | `/api/v1/tasks/{id}/complete` | Mark task as `COMPLETED` | `200 OK` |
| `DELETE` | `/api/v1/tasks/{id}` | Permanently delete a task | `204 No Content` |

---

## Running Automated Tests

Run the complete test suite with the Maven wrapper:
```bash
# Windows
.\mvnw.cmd test -o

# Linux / macOS
./mvnw test -o
```

---

## Documentation

Detailed technical documentation is available in the [`docs/`](docs/) directory:
- [PRD.md](docs/PRD.md): Product Requirements Document
- [SRS.md](docs/SRS.md): Software Requirements Specification
- [ARCHITECTURE.md](docs/ARCHITECTURE.md): Architecture & Design Patterns
- [API_DESIGN.md](docs/API_DESIGN.md): REST API Specifications & Contracts
- [DATABASE_DESIGN.md](docs/DATABASE_DESIGN.md): PostgreSQL Schema & Indexing
- [TESTING_STRATEGY.md](docs/TESTING_STRATEGY.md): Testing Pyramid & Quality Standards
- [GIT_WORKFLOW.md](docs/GIT_WORKFLOW.md): Contribution & Branching Guide
