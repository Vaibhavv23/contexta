# ARCHITECTURE.md — Planned V1 Architecture

Status: planning. This is the **initial V1 architecture**, not a rigid commitment.
Revisit via an ADR in [DECISIONS.md](./DECISIONS.md) when a requirement genuinely
demands a change. Nothing here is implemented yet.

Related: [PROJECT_MEMORY.md](../PROJECT_MEMORY.md), [DATA_MODEL.md](./DATA_MODEL.md),
[AI_SYSTEM.md](./AI_SYSTEM.md), [INTEGRATIONS.md](./INTEGRATIONS.md),
[SECURITY.md](./SECURITY.md), [DEVELOPMENT.md](./DEVELOPMENT.md).

---

## Conceptual Diagram

```text
                    Next.js (web app)
                       │
                       ▼
                  FastAPI API
                       │
          ┌────────────┼────────────┐
          │            │            │
          ▼            ▼            ▼
     PostgreSQL      Redis      LLM APIs
          │       (queue +       (hosted)
          │        cache)
          ▼
      pgvector
          │
          ▼
   Investigation Engine
          │
     ┌────┼─────┐
     ▼    ▼     ▼
   Slack Jira GitHub   (read-only connectors)
```

## Shape: Modular Monolith

One deployable backend process (plus one or more worker processes sharing the same
codebase). Modules are separated by clear internal boundaries and packages, not by
network calls. Rationale in ADR-001 / ADR-004.

Backend module boundaries (planned Python packages):

```text
app/
  core/           config, logging, db session, security primitives
  auth/           authentication, sessions, current-user
  tenancy/        organization / membership scoping helpers
  connections/    OAuth flows, encrypted token store, connection lifecycle
  sync/           sync orchestration, job definitions, cursors
  connectors/
    slack/
    jira/
    github/
  ingest/         normalization into the unified model, chunking, embedding
  model/          unified data model, entity resolution, relationships
  search/         keyword search, vector search, hybrid retrieval, reranking
  evidence/       evidence item creation and provenance
  investigation/  planner, orchestrator, timeline, reasoning, report assembly
  agent/          tool definitions and the bounded tool-use loop
  permissions/    permission-aware retrieval filter
  api/            FastAPI routers (thin; delegate to modules)
  observability/  tracing, metrics, cost accounting
```

## Frontend Architecture

- Next.js (App Router) + TypeScript + Tailwind.
- Server-side auth check; session cookie issued by the backend.
- Pages (planned): connections/setup, new investigation, investigation detail (streamed
  progress, timeline, report with expandable evidence and source links), investigations list.
- Talks only to the FastAPI backend over `/api/v1`. No direct calls to Slack/Jira/GitHub.
- Streaming of investigation progress via SSE or chunked responses (decide at Phase 22).

## Backend Architecture

- Python + FastAPI, ASGI (uvicorn/gunicorn).
- Thin routers under `/api/v1`; business logic in modules.
- Pydantic models for request/response; typed service layer.
- Synchronous request handlers for CRUD; long work (sync, investigations) dispatched to
  the worker via Redis and reported back through job records + streaming.
- SQLAlchemy (or equivalent) + Alembic migrations.

## Database

- PostgreSQL is the single system of record (ADR-002).
- `pgvector` extension in the same instance for embeddings (ADR-003).
- Every domain table carries `organization_id`; all queries filter by it.
- Raw source payloads retained (JSONB) alongside normalized columns for reprocessing.
- Migrations via Alembic; no manual schema edits.

## Background Processing

- Redis as broker + lightweight cache.
- A worker process (RQ / Arq / Celery — decide at Phase 5) consuming jobs:
  - initial sync, incremental sync (per connection, per resource)
  - embedding / re-embedding
  - investigation runs
- Jobs are idempotent and resumable via stored cursors/watermarks.
- Job state persisted in `sync_jobs` / an `investigations` status field for observability.

## Integrations

- One connector package per source, each exposing: `authorize()`, `exchange_code()`,
  `refresh()`, `list_*()` / `fetch_*()` with pagination, and `to_normalized()` mappers.
- Connectors are read-only.
- Rate limiting and retry/backoff handled in a shared connector base.
- All external API specifics verified against official docs at implementation time
  (Rule 8). See [INTEGRATIONS.md](./INTEGRATIONS.md).

## Search

- Keyword: PostgreSQL full-text search (`tsvector` / GIN) over normalized text.
- Semantic: pgvector similarity over chunk embeddings.
- Hybrid: run both, fuse (e.g. reciprocal rank fusion), then rerank.
- All search paths apply tenant + permission + metadata filters before returning.

## RAG

- Per-source chunking with metadata (system, object type, ids, author, timestamp,
  channel/project/repo, URL).
- Embeddings stored in pgvector with the metadata needed for filtering.
- Explicit, size-bounded context construction for the LLM; context is assembled only
  from authorized evidence items. See [AI_SYSTEM.md](./AI_SYSTEM.md).

## Evidence Engine

- Converts a retrieved record/chunk into an `evidence` row: source reference, quoted
  span, author, timestamp, retrieval reason, and an evidence level
  (Confirmed / Likely / Possible / Unknown).
- Evidence items are immutable within an investigation and referenced by id in the report.

## Investigation Engine

- Orchestrates: plan → gather (via agent tools / retrieval) → cross-reference → timeline
  → hypotheses → reasoning → report.
- Deterministic control flow in V1 (ADR-005); the LLM fills in steps, it does not choose
  the overall procedure.
- Bounded: maximum tool iterations, maximum evidence items, maximum tokens/cost.
- Produces both structured data and prose.

## Authentication

- Email/password (or magic link — decide at Phase 2) issuing a signed session cookie.
- No third-party SSO in V1.
- `current_user` dependency resolves the user and their memberships on every request.

## Multi-Tenancy

- Tenant = `organization`. `organization_id` on every domain row.
- A `tenancy` helper enforces scoping; services never issue unscoped queries.
- Consider PostgreSQL RLS as defense-in-depth later (open question, see PROJECT_MEMORY §30).

## Security

- OAuth tokens encrypted at rest (app-level envelope encryption; key from secrets manager).
- Secrets only from environment / secrets manager, never committed, never logged.
- Permission-aware retrieval sits between retrieval and the LLM:
  `User → Identity → Authorization → permission-aware retrieval → authorized evidence → LLM`.
- Audit log for connection changes, investigations, and data access.
- Full model in [SECURITY.md](./SECURITY.md).

## Observability

- Structured JSON logging with request/job/investigation correlation ids.
- Metrics: sync throughput and lag, search latency, investigation duration, tool-call
  counts, token usage and cost per investigation.
- Tracing across API → worker → LLM calls.
- Per-investigation cost record for the cost controls in Phase 27.

## Deployment (V1)

- Containerized backend + worker, managed PostgreSQL (with pgvector), managed Redis.
- Single region. No Kubernetes required for V1 — a simple container platform is enough.
- Details decided at Phase 28.

## Explicitly Out of Scope for V1 Architecture

Kafka / event bus, Neo4j / graph DB, microservices, service mesh, multi-region,
self-hosted models, CQRS/event sourcing. Add only with an ADR.
