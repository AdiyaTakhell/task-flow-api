# TaskFlow API — API Design Document

**Document Version:** 1.2  
**Status:** Approved  
**Base Path:** `/api/v1`  
**Content-Type:** `application/json`  
**OpenAPI Specification:** OpenAPI 3.0 via `springdoc-openapi` (`/v3/api-docs`, `/swagger-ui/index.html`)  

---

## 1. REST API Conventions

### 1.1 HTTP Verbs
- `POST`: Create a new resource.
- `GET`: Retrieve a resource or collection (safe, idempotent).
- `PUT`: Update an existing resource details completely (idempotent).
- `PATCH`: Apply a partial mutation or state-transition action (e.g., mark as completed).
- `DELETE`: Remove a resource permanently (idempotent).

### 1.2 HTTP Status Codes
| Status Code | Reason Phrase | Application Context |
| :--- | :--- | :--- |
| `200 OK` | OK | Successful GET, PUT, or PATCH operation. |
| `201 Created` | Created | Successful POST creation. Includes `Location` header. |
| `204 No Content` | No Content | Successful DELETE. Response body is empty. |
| `400 Bad Request` | Bad Request | Request payload or parameter validation failure (`ProblemDetail`). |
| `404 Not Found` | Not Found | Requested resource ID does not exist (`ProblemDetail`). |
| `500 Internal Server Error` | Server Error | Unhandled server error with sanitized detail (`ProblemDetail`). |
| `503 Service Unavailable` | Service Unavailable | System or database connectivity down (`HealthResponse`). |

### 1.3 Correlation Header
All responses include the `X-Request-ID` header. If supplied in the request, it is echoed back; otherwise, a UUID is automatically generated:
```http
X-Request-ID: 7b9f31a2-8b40-4f51-b8ef-5e362fa7d109
```

### 1.4 Standard Error Response Schema (RFC 7807 / RFC 9457 ProblemDetail)
All 4xx and 5xx errors return a standardized `ProblemDetail` payload:

```json
{
  "type": "urn:problem:validation-failed",
  "title": "Validation Failed",
  "status": 400,
  "detail": "Request validation failed on one or more fields",
  "timestamp": "2026-11-13T22:00:00Z",
  "invalidParams": {
    "title": "Title is required and cannot be blank"
  }
}
```

For missing resources (404):
```json
{
  "type": "urn:problem:task-not-found",
  "title": "Task Not Found",
  "status": 404,
  "detail": "Task not found with ID: 999",
  "timestamp": "2026-11-13T22:00:00Z",
  "taskId": 999
}
```

---

## 2. API Endpoints Catalog

### 2.1 Health Check
Verifies application status and database connectivity.

* **Endpoint:** `GET /api/v1/health`
* **Response (Healthy):** `200 OK`
```json
{
  "status": "UP",
  "service": "taskflow-api",
  "timestamp": "2026-11-13T22:30:00Z",
  "details": {
    "database": "UP"
  }
}
```
* **Response (Degraded/Down):** `503 Service Unavailable`
```json
{
  "status": "DOWN",
  "service": "taskflow-api",
  "timestamp": "2026-11-13T22:30:00Z",
  "details": {
    "database": "DOWN"
  }
}
```

---

### 2.2 Create Task
Creates a new task in `PENDING` status.

* **Endpoint:** `POST /api/v1/tasks`
* **Request Headers:** `Content-Type: application/json`
* **Request Body:**
```json
{
  "title": "Build REST API endpoints",
  "description": "Implement CRUD controllers, DTOs, and error handlers",
  "priority": "HIGH",
  "dueDate": "2026-12-31"
}
```
* **Field Rules:**
  - `title` *(String, Required)*: 1 to 150 characters, non-blank.
  - `description` *(String, Optional)*: Free-form text.
  - `priority` *(String, Optional)*: `"LOW"`, `"MEDIUM"`, `"HIGH"`. Defaults to `"MEDIUM"`.
  - `dueDate` *(String, Optional)*: Format `YYYY-MM-DD`.

* **Success Response:** `201 Created`  
  **Headers:** `Location: /api/v1/tasks/1`
```json
{
  "id": 1,
  "title": "Build REST API endpoints",
  "description": "Implement CRUD controllers, DTOs, and error handlers",
  "status": "PENDING",
  "priority": "HIGH",
  "dueDate": "2026-12-31",
  "createdAt": "2026-11-13T14:30:00Z",
  "updatedAt": "2026-11-13T14:30:00Z"
}
```

---

### 2.3 List Tasks (Pagination, Filtering, Sorting)
Retrieves a paginated list of tasks matching optional multi-criteria filters and sort order.

* **Endpoint:** `GET /api/v1/tasks`
* **Query Parameters:**
  - `status` *(String, Optional)*: `"PENDING"` or `"COMPLETED"`.
  - `priority` *(String, Optional)*: `"LOW"`, `"MEDIUM"`, `"HIGH"`.
  - `dueDate` *(String, Optional)*: Format `YYYY-MM-DD`. Exact date match.
  - `title` *(String, Optional)*: Case-insensitive keyword search against task title.
  - `page` *(Integer, Optional)*: Zero-based page index (default: `0`, minimum: `0`).
  - `size` *(Integer, Optional)*: Page size (default: `10`, minimum: `1`, maximum: `100`).
  - `sort` *(String, Optional)*: Sort expression in `<property>,<direction>` format (default: `createdAt,desc`). Allowed properties: `id`, `title`, `status`, `priority`, `dueDate`, `createdAt`, `updatedAt`. Direction: `asc` or `desc`.
* **Success Response:** `200 OK`
```json
{
  "content": [
    {
      "id": 1,
      "title": "Build REST API endpoints",
      "description": "Implement CRUD controllers, DTOs, and error handlers",
      "status": "PENDING",
      "priority": "HIGH",
      "dueDate": "2026-12-31",
      "createdAt": "2026-11-13T14:30:00Z",
      "updatedAt": "2026-11-13T14:30:00Z"
    }
  ],
  "page": 0,
  "size": 10,
  "totalElements": 1,
  "totalPages": 1,
  "first": true,
  "last": true
}
```
* **Error Response:** `400 Bad Request` (RFC 7807/9457 `ProblemDetail`) when passing invalid query parameter types or unapproved sort properties.

---

### 2.4 Retrieve Task by ID
Fetches a single task by its unique numeric identifier.

* **Endpoint:** `GET /api/v1/tasks/{id}`
* **Path Parameters:**
  - `id` *(Long, Required)*: Positive integer.
* **Success Response:** `200 OK`
```json
{
  "id": 1,
  "title": "Build REST API endpoints",
  "description": "Implement CRUD controllers, DTOs, and error handlers",
  "status": "PENDING",
  "priority": "HIGH",
  "dueDate": "2026-12-31",
  "createdAt": "2026-11-13T14:30:00Z",
  "updatedAt": "2026-11-13T14:30:00Z"
}
```
* **Error Response:** `404 Not Found` (ProblemDetail)

---

### 2.5 Update Task Details
Replaces editable attributes of an existing task.

* **Endpoint:** `PUT /api/v1/tasks/{id}`
* **Request Body:**
```json
{
  "title": "Refactor REST API endpoints",
  "description": "Add pagination and filtering",
  "priority": "HIGH",
  "dueDate": "2026-12-31"
}
```
* **Success Response:** `200 OK`
```json
{
  "id": 1,
  "title": "Refactor REST API endpoints",
  "description": "Add pagination and filtering",
  "status": "PENDING",
  "priority": "HIGH",
  "dueDate": "2026-12-31",
  "createdAt": "2026-11-13T14:30:00Z",
  "updatedAt": "2026-11-13T15:00:00Z"
}
```

---

### 2.6 Mark Task as Completed
Transitions task status to `COMPLETED`.

* **Endpoint:** `PATCH /api/v1/tasks/{id}/complete`
* **Success Response:** `200 OK`
```json
{
  "id": 1,
  "title": "Refactor REST API endpoints",
  "description": "Add pagination and filtering",
  "status": "COMPLETED",
  "priority": "HIGH",
  "dueDate": "2026-12-31",
  "createdAt": "2026-11-13T14:30:00Z",
  "updatedAt": "2026-11-13T15:10:00Z"
}
```

---

### 2.7 Delete Task
Permanently deletes a task by ID.

* **Endpoint:** `DELETE /api/v1/tasks/{id}`
* **Success Response:** `204 No Content` (Empty body)
* **Error Response:** `404 Not Found` (ProblemDetail)
