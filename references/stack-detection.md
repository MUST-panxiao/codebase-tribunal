# Stack Detection Protocol

Shared reference for all personas. The review must detect the project's stack and
conventions before any role starts reviewing — never assume a stack.

## Goal

Produce a one-line stack summary before any review work begins. Example:

> `Stack: Python (FastAPI) + TypeScript (Vue 3) · SQLite · Docker · pytest — root: /path/to/project`

## Procedure

### 1. Find the project root

Walk up from the target directory looking for the nearest directory that has a manifest
file and/or a `.git` directory:

**Manifests to look for:**
- `package.json` → Node.js / JavaScript / TypeScript
- `pyproject.toml` / `setup.py` / `requirements.txt` → Python
- `go.mod` → Go
- `Cargo.toml` → Rust
- `pom.xml` / `build.gradle` → Java
- `*.csproj` → C# / .NET
- `Gemfile` → Ruby
- `composer.json` → PHP

### 2. Read project instructions

If the root has any of these, read them — project conventions override skill defaults:
- `CLAUDE.md` / `AGENTS.md` — AI assistant instructions
- `.cursorrules` — Cursor IDE rules
- `CONTRIBUTING.md` — contribution guidelines
- `README.md` — project overview

### 3. Read the manifest(s)

Identify:
- **Language:** Python / TypeScript / Go / Rust / Java / ...
- **Framework:** FastAPI / Django / Vue / React / Express / Gin / ...
- **Key dependencies:** ORM, auth library, UI component library, test framework
- **Version constraints:** pinned versions, floating ranges

For monorepos, read both the root manifest and the workspace/package manifest
closest to the files under review.

### 4. Identify package manager / build tool

From the lockfile:
- `pnpm-lock.yaml` → pnpm
- `yarn.lock` → yarn
- `package-lock.json` → npm
- `bun.lockb` → bun
- `poetry.lock` / `uv.lock` → Poetry / uv
- `requirements.txt` → pip
- `go.sum` → go modules
- `Cargo.lock` → cargo

### 5. Scan the top-level layout

Learn the project's directory conventions. Note:
- Where does source code live? (`src/`, `app/`, `cmd/`, `lib/`)
- Where do tests live? (`tests/`, `__tests__/`, `*_test.go`)
- Where is configuration? (`configs/`, `config/`, `.env.example`)
- Is there a clear separation of frontend and backend?

### 6. Detect deployment artifacts

Check for:
- `Dockerfile` / `docker-compose.yml` → Docker deployment
- `.github/workflows/` → GitHub Actions CI/CD
- `.gitlab-ci.yml` → GitLab CI
- `Makefile` → build automation
- `vercel.json` / `netlify.toml` → platform deployment
- `*.tf` → Terraform IaC
- `helm/` → Kubernetes Helm chart

### 7. Detect test framework

From manifest and config files:
- `pytest` / `unittest` → Python
- `jest` / `vitest` / `mocha` → JavaScript/TypeScript
- `go test` → Go
- `cargo test` → Rust
- `junit` / `testng` → Java
- `rspec` → Ruby

Note: e2e frameworks (`playwright`, `cypress`, `selenium`) if present.

### 8. Detect database / data layer

From manifest and config:
- ORM: SQLAlchemy / Prisma / Drizzle / GORM / TypeORM / Sequelize
- Database: PostgreSQL / MySQL / SQLite / MongoDB / Redis
- Migration tool: Alembic / Prisma Migrate / goose / Flyway

## Output

State the detected stack in one line. If detection fails, say so — do not guess.

**Full-stack example:**
> `Stack: Python 3.12 (FastAPI 0.115, SQLAlchemy 2.0 async, aiosqlite) + TypeScript (Vue 3, Element Plus, ECharts, Pinia) · SQLite · Docker (single container) · No test framework — root: /path/to/project`

**Frontend-only example:**
> `Stack: TypeScript (React 18, Next.js 14 App Router, Tailwind CSS, shadcn/ui) · Vercel deployment · Vitest + Playwright — root: /path/to/project`

**Python-only example:**
> `Stack: Python 3.11 (Django 5.0, DRF, Celery) · PostgreSQL · Redis · Docker Compose · pytest — root: /path/to/project`

## Rules

- Detected facts always beat assumptions. If the manifest says Vue 2 but you "feel" it's Vue 3,
  trust the manifest.
- If the project has multiple manifests (e.g., separate frontend/ and backend/ directories),
  detect both stacks and note them separately.
- If cwd has no manifest but contains multiple child repos: STOP. Ask user which repo/path to review.
- Re-detect every invocation — don't cache between reviews.
