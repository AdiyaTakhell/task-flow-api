# TaskFlow API — Git Workflow & Contribution Guide

**Document Version:** 1.0  
**Status:** Approved  
**Repository Model:** GitHub Standard Feature Branch Workflow  
**Commit Standard:** Conventional Commits v1.0.0  

---

## 1. Branching Strategy

The repository maintains a simple, stable branching model:

```
main ─────────────────────────●──────────────────────────●────── (Production Ready)
      \                      /                          /
       └── feature/create-task ── fix/due-date-validation
```

### 1.1 Branch Taxonomy
| Branch | Role | Rules |
| :--- | :--- | :--- |
| `main` | Production-ready baseline | Must always build cleanly and pass 100% of tests. Direct pushes are forbidden. |
| `feature/<name>` | New capabilities / endpoints | Branched from `main`. Short-lived (1-3 days). |
| `fix/<name>` | Defect and bug corrections | Branched from `main`. Addresses an issue or test failure. |
| `test/<name>` | Automated test additions | Branched from `main`. Dedicated to test coverage. |
| `docs/<name>` | Documentation updates | Updates to `/docs` or `README.md`. |
| `refactor/<name>` | Code restructuring | Non-behavioral improvements. |

---

## 2. Commit Message Conventions

All commit messages must adhere to the **Conventional Commits** specification:

```
<type>(<scope>): <short imperative description>

[optional body explaining motivation and context]

[optional footer: e.g., Closes #12]
```

### 2.1 Types
- `feat`: A new feature or endpoint.
- `fix`: A bug fix or validation correction.
- `test`: Adding or correcting tests.
- `docs`: Documentation only changes.
- `refactor`: Code change that neither fixes a bug nor adds a feature.
- `build`: Changes that affect the build system or dependencies (`pom.xml`).
- `chore`: Maintenance tasks, `.gitignore` updates, formatting.

### 2.2 Examples
```bash
feat(task): add task creation endpoint with validation
fix(task): enforce UTC timezone on updated_at timestamp
test(task): add Testcontainers repository integration test
docs(api): document query parameters for task list endpoint
build(deps): update PostgreSQL driver to latest version
```

---

## 3. Step-by-Step Developer Workflow

### Step 1: Synchronize Baseline
Always branch from an up-to-date `main`:
```bash
git switch main
git pull origin main
```

### Step 2: Create a Dedicated Feature Branch
```bash
git switch -c feature/create-task
```

### Step 3: Implement & Test Locally
Implement the required classes following the layered architecture. Before staging changes, run the full automated verification:
```bash
# On Linux / macOS:
./mvnw clean test

# On Windows PowerShell:
.\mvnw.cmd clean test
```

### Step 4: Review and Stage Changes
```bash
git status
git diff
git add src/ docs/
```

### Step 5: Commit Changes
```bash
git commit -m "feat(task): implement create task endpoint and unit tests"
```

### Step 6: Merge or Open a Pull Request
For solo workflows:
```bash
git switch main
git merge --no-ff feature/create-task
git push origin main
git branch -d feature/create-task
```

For team / GitHub workflows:
```bash
git push -u origin feature/create-task
```
Then open a Pull Request against `main` on GitHub.

---

## 4. Repository Structure & Configuration

```
taskflow-api/
├── .github/
│   └── pull_request_template.md
├── docs/
│   ├── PRD.md
│   ├── SRS.md
│   ├── DATABASE_DESIGN.md
│   ├── API_DESIGN.md
│   ├── ARCHITECTURE.md
│   ├── GIT_WORKFLOW.md
│   └── TESTING_STRATEGY.md
├── src/
│   ├── main/
│   │   ├── java/com/taskflow/...
│   │   └── resources/
│   │       ├── application.yml
│   │       └── db/migration/V1__create_tasks_table.sql
│   └── test/
├── docker-compose.yml
├── .gitignore
├── mvnw
├── mvnw.cmd
├── pom.xml
└── README.md
```

### 4.1 Recommended `.gitignore`

```gitignore
# Maven build target
target/
pom.xml.tag
pom.xml.releaseBackup
pom.xml.versionsBackup

# IntelliJ IDEA
.idea/
*.iml
*.iws
out/

# Eclipse
.classpath
.project
.settings/

# Visual Studio Code
.vscode/

# Logs and runtime data
*.log
logs/

# Environment files
.env
.env.*

# OS generated artifacts
.DS_Store
Thumbs.db
```

---

## 5. Pull Request Quality Checklist

Before approving or merging any branch into `main`:
- [ ] Branch is focused on a single responsibility.
- [ ] Code follows package-by-feature and layered architecture rules.
- [ ] No entities are exposed in controller contracts or DTOs.
- [ ] Input validation annotations (`@Valid`, `@NotBlank`, etc.) are in place.
- [ ] `./mvnw clean test` passes with zero errors and zero warnings.
- [ ] Flyway migrations run cleanly against real PostgreSQL.
- [ ] No hardcoded passwords, tokens, or environment credentials are committed.
- [ ] Commit history is clean with descriptive Conventional Commit messages.
