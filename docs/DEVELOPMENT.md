# DEVELOPMENT.md — Development Workflow & Standards

Status: planning. The toolchain is not installed. Do not install or configure the full
stack until a task explicitly requires it (Phase 1 onward).

Related: [ARCHITECTURE.md](./ARCHITECTURE.md), [DECISIONS.md](./DECISIONS.md),
[../CLAUDE.md](../CLAUDE.md).

---

## Local Development

Planned stack:

```text
Frontend:  Next.js, TypeScript, Tailwind
Backend:   Python, FastAPI
Database:  PostgreSQL
Vector:    pgvector (extension in the same PostgreSQL instance)
Queue:     Redis + worker process
AI:        hosted LLM API (embeddings + generation)
Testing:   backend (pytest), frontend (vitest/jest + React Testing Library),
           integration tests against a test database
```

Planned layout:

```text
frontend/   Next.js app
backend/    FastAPI app + worker + Alembic migrations
tests/      cross-cutting / integration tests and fixtures
scripts/    dev and ops scripts
docs/       documentation
tasks/      roadmap and current task
```

Planned local orchestration: a `docker-compose` for PostgreSQL (with pgvector) and Redis;
backend and frontend run on the host during development. Introduced at Phase 1.

Environment: copy `.env.example` to `.env` and fill in values. `.env` is git-ignored.

---

## Coding Standards

**TypeScript**
- `strict: true`. No implicit `any`. No non-null assertions except with a comment.
- Prefer explicit return types on exported functions.
- ESLint + Prettier; CI fails on lint or format errors.

**Python**
- Type hints on all function signatures; `mypy` (or `pyright`) in CI.
- `ruff` for lint + import sorting; `black` for formatting; CI fails on violations.
- Pydantic for I/O models; dataclasses/plain classes for internal types.

**API naming**
- REST under `/api/v1`. Resource-oriented, plural nouns: `/api/v1/connections`,
  `/api/v1/investigations`, `/api/v1/investigations/{id}/events`.
- snake_case JSON fields. ISO-8601 UTC timestamps. UUID ids.
- Consistent pagination (`limit` + cursor) and error envelope.

**Error handling**
- Backend: typed exceptions mapped to HTTP responses by a single handler; never leak
  stack traces or secrets to clients.
- External API calls: explicit timeouts, retry with backoff, and a clear failure state
  recorded on the job/connection.
- Frontend: distinguish expected (validation, auth) from unexpected errors; user-facing
  messages never expose internals.

**Logging**
- Structured JSON. Correlation ids: `request_id`, `job_id`, `investigation_id`.
- Respect the logging allowlist in [SECURITY.md](./SECURITY.md). Never log secrets,
  tokens, or full source-content bodies.
- Levels: DEBUG (dev only), INFO (lifecycle), WARNING (recoverable), ERROR (needs
  attention).

**Environment variables**
- All config via env; documented in `.env.example`.
- Loaded through one typed settings object per app (no scattered `os.environ` reads).
- No secrets in code, tests, fixtures, or git history.

**Testing**
- Every meaningful backend feature: unit tests + at least one integration test against a
  test PostgreSQL.
- Every meaningful frontend feature: component/behavior tests; critical flows get an
  end-to-end test later.
- Connectors: tests use recorded fixtures / mocked HTTP, never live third-party calls in
  CI.
- Multi-tenancy: tests that assert cross-tenant isolation for each data-access path.
- A task is not "done" until its tests exist and pass (Rule 4).

**Formatting & linting**
- Enforced in CI and via pre-commit hooks (set up at Phase 1).

**Commit conventions**
- Conventional Commits: `feat:`, `fix:`, `chore:`, `docs:`, `test:`, `refactor:`.
- One logical change per commit. Reference the roadmap task.
- Commit/push only when the user asks. Branch off `main` for work.

---

## Feature Workflow

```text
Task        pick the next unchecked item from tasks/TODO.md; write tasks/CURRENT_TASK.md
 ↓
Inspect     read PROJECT_MEMORY.md, relevant docs/, and the existing code
 ↓
Plan        affected files, approach, risks (in CURRENT_TASK.md)
 ↓
Implement   smallest change that satisfies the task
 ↓
Test        write and run tests; do not proceed on failures
 ↓
Review      self-review the diff for scope creep, security, style
 ↓
Document    update PROJECT_MEMORY.md, docs/, tasks/TODO.md
 ↓
Commit      Conventional Commit; archive CURRENT_TASK.md to tasks/completed/
```

---

## Definition of Done (per task)

- [ ] Code is the minimal change for the task.
- [ ] Tests written and passing (backend and/or frontend as appropriate).
- [ ] Lint, format, and type checks pass.
- [ ] No secrets added; logging allowlist respected.
- [ ] Multi-tenancy scoping verified for any new data-access path.
- [ ] Docs updated (`PROJECT_MEMORY.md`, relevant `docs/`).
- [ ] `tasks/TODO.md` checkbox checked; `tasks/CURRENT_TASK.md` result filled in and
      archived to `tasks/completed/`.
