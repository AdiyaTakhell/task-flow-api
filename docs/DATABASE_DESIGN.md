# TaskFlow API — Database Design Document

**Document Version:** 1.1  
**Status:** Approved  
**Database Engine:** PostgreSQL 18-alpine  
**Schema Migration Tool:** Flyway 10+  
**ORM / Persistence:** Hibernate 7.x / Spring Data JPA  

---

## 1. Overview & Architectural Principles

The TaskFlow API database design prioritizes **data integrity, query predictability, and schema evolution control**.

Core principles:
- **Strict Typing:** Leverage native PostgreSQL data types (`TIMESTAMPTZ`, `BIGINT`, `DATE`, `TEXT`).
- **Database-Enforced Integrity:** Never rely solely on application-layer validation. Enforce constraints (`NOT NULL`, `CHECK`) directly in SQL.
- **Controlled Migrations:** All schema modifications must occur via immutable Flyway migration scripts. Hibernate's `ddl-auto` is strictly set to `validate`.
- **Query-Driven Indexing:** Indexes are created strictly to satisfy specified access patterns (pagination, composite filtering, ordering).

---

## 2. Relational Schema & Data Dictionary

### 2.1 Table: `tasks`

```
                                  Table "public.tasks"
   Column    |           Type           | Collation | Nullable |             Default              
-------------+--------------------------+-----------+----------+----------------------------------
 id          | bigint                   |           | not null | generated always as identity
 title       | character varying(150)   |           | not null | 
 description | text                     |           |          | 
 status      | character varying(20)    |           | not null | 'PENDING'::character varying
 priority    | character varying(20)    |           | not null | 'MEDIUM'::character varying
 due_date    | date                     |           |          | 
 created_at  | timestamp with time zone |           | not null | CURRENT_TIMESTAMP
 updated_at  | timestamp with time zone |           | not null | CURRENT_TIMESTAMP
Indexes:
    "pk_tasks" PRIMARY KEY, btree (id)
    "idx_tasks_status_created" btree (status, created_at DESC)
    "idx_tasks_due_date" btree (due_date)
Check constraints:
    "chk_tasks_priority" CHECK (priority::text = ANY (ARRAY['LOW'::character varying, 'MEDIUM'::character varying, 'HIGH'::character varying]::text[]))
    "chk_tasks_status" CHECK (status::text = ANY (ARRAY['PENDING'::character varying, 'COMPLETED'::character varying]::text[]))
    "chk_tasks_title_not_blank" CHECK (length(btrim(title::text)) > 0)
```

### 2.2 Column Specifications & Java Mappings

| Column Name | SQL Type | Nullable | Default | Java Attribute | Java Type | Description |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| `id` | `BIGINT GENERATED ALWAYS AS IDENTITY` | No | Auto-gen | `id` | `Long` | Primary key |
| `title` | `VARCHAR(150)` | No | None | `title` | `String` | Task headline (1-150 chars) |
| `description` | `TEXT` | Yes | `NULL` | `description` | `String` | Free-form task details |
| `status` | `VARCHAR(20)` | No | `'PENDING'` | `status` | `TaskStatus` | Enum: `PENDING`, `COMPLETED` |
| `priority` | `VARCHAR(20)` | No | `'MEDIUM'` | `priority` | `TaskPriority` | Enum: `LOW`, `MEDIUM`, `HIGH` |
| `due_date` | `DATE` | Yes | `NULL` | `dueDate` | `LocalDate` | Optional calendar target date |
| `created_at` | `TIMESTAMPTZ` | No | `CURRENT_TIMESTAMP` | `createdAt` | `Instant` | UTC creation timestamp |
| `updated_at` | `TIMESTAMPTZ` | No | `CURRENT_TIMESTAMP` | `updatedAt` | `Instant` | UTC last updated timestamp |

---

## 3. Key Design Decisions

### 3.1 Primary Key Strategy
- **Decision:** Use `BIGINT GENERATED ALWAYS AS IDENTITY`.
- **Rationale:**
  - Standard SQL compliance (replaces legacy PostgreSQL `SERIAL` / `BIGSERIAL`).
  - Guarantees 64-bit integer headroom, preventing integer overflow.
  - Aligns cleanly with Hibernate's `GenerationType.IDENTITY`.

### 3.2 Timestamp Handling & Timezone Safety
- **Decision:** Use `TIMESTAMPTZ` (`TIMESTAMP WITH TIME ZONE`).
- **Rationale:**
  - PostgreSQL stores all `TIMESTAMPTZ` internally in UTC.
  - Prevents timezone skew and daylight-saving shifts when accessed across clients in different geographical locations.
  - Mapped directly to Java 8+ `java.time.Instant`.
  - Managed in JPA via `@CreationTimestamp` and `@UpdateTimestamp` annotations.

### 3.3 Check Constraints vs PostgreSQL Enums
- **Decision:** Store status and priority as `VARCHAR(20)` with SQL `CHECK` constraints rather than native PostgreSQL `CREATE TYPE ... AS ENUM`.
- **Rationale:**
  - Native PostgreSQL enum types introduce migration friction when adding new values in distributed/replicated environments (e.g. `ALTER TYPE ADD VALUE` cannot run inside certain transaction blocks in older PostgreSQL versions).
  - `VARCHAR` with `CHECK` constraints provides identical validation guarantees while allowing seamless Flyway migration updates.

---

## 4. Indexing & Query Optimization Strategy

### 4.1 Index Analysis

#### 1. Composite Index: `idx_tasks_status_created`
```sql
CREATE INDEX idx_tasks_status_created ON tasks (status, created_at DESC);
```
- **Target Query:** `SELECT * FROM tasks WHERE status = 'PENDING' ORDER BY created_at DESC LIMIT 10 OFFSET 0;`
- **Why it matters:** Low-cardinality columns like `status` (which has only 2 distinct values) make single-column B-tree indexes inefficient. By indexing `(status, created_at DESC)`, PostgreSQL can satisfy the filter **and** return records in the requested sort order directly from the index scan without an expensive in-memory sort (`Sort` step eliminated from query plan).

#### 2. Single-Column Index: `idx_tasks_due_date`
```sql
CREATE INDEX idx_tasks_due_date ON tasks (due_date);
```
- **Target Query:** Filtering on deadlines (`WHERE due_date = '2026-11-20'`) and future range scans (`WHERE due_date <= CURRENT_DATE`).

#### 3. Title Substring Searches (`ILIKE '%keyword%'`)
- **Query:** `SELECT * FROM tasks WHERE LOWER(title) LIKE LOWER('%keyword%');`
- **Performance Note:** Standard B-tree indexes cannot optimize leading-wildcard queries (`%keyword%`). For v1 task volumes (< 100,000 rows), a sequential table scan is extremely fast in PostgreSQL memory cache. For future high-scale search, enable the `pg_trgm` extension:
  ```sql
  -- Future optimization if needed:
  -- CREATE EXTENSION IF NOT EXISTS pg_trgm;
  -- CREATE INDEX idx_tasks_title_trgm ON tasks USING gin (title gin_trgm_ops);
  ```

---

## 5. Flyway Migration Strategy

### 5.1 Migration File Conventions
All migrations reside in `src/main/resources/db/migration/` and follow the strict naming pattern:
```
V<Version>__<Description>.sql
```
Example: `V1__create_tasks_table.sql`

### 5.2 Initial Migration Script (`V1__create_tasks_table.sql`)

```sql
-- V1__create_tasks_table.sql
-- Create initial tasks table and related indexes

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

-- Index for combined status filtering and default created_at descending sort
CREATE INDEX idx_tasks_status_created ON tasks (status, created_at DESC);

-- Index for due date filtering
CREATE INDEX idx_tasks_due_date ON tasks (due_date);
```

### 5.3 Spring Boot Configuration (`application.yml`)
```yaml
spring:
  datasource:
    url: jdbc:postgresql://${DB_HOST:localhost}:${DB_PORT:5432}/${DB_NAME:taskflow_db}
    username: ${DB_USER:taskflow}
    password: ${DB_PASSWORD:taskflow_secret}
    driver-class-name: org.postgresql.Driver
  jpa:
    hibernate:
      ddl-auto: validate
    properties:
      hibernate:
        dialect: org.hibernate.dialect.PostgreSQLDialect
        format_sql: true
    open-in-view: false
  flyway:
    enabled: true
    baseline-on-migrate: true
    locations: classpath:db/migration
```
