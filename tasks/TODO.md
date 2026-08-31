# V1 Implementation Roadmap

Complete checklist for V1 of the Enterprise Context & Investigation Agent.

**Rules:**
- Do exactly one task at a time. See [../CLAUDE.md](../CLAUDE.md).
- Only check a box when the task is implemented **and tested** **and documented**.
- Do not check future implementation boxes. Only Phase 0 documentation is done.
- Discovered work goes here as new checkboxes; it does not silently expand the current task.

Legend: `[x]` done · `[ ]` not started.

---

## PHASE 0 — Product & Architecture Foundation

- [x] Create repository documentation structure (`docs/`, `tasks/`, `tests/`, `scripts/`)
- [x] Write `CLAUDE.md` (role, rules, docs to consult)
- [x] Write `PROJECT_MEMORY.md` (full project memory, current state = initialization)
- [x] Write `docs/PRODUCT.md`
- [x] Write `docs/ARCHITECTURE.md`
- [x] Write `docs/DATA_MODEL.md`
- [x] Write `docs/INTEGRATIONS.md`
- [x] Write `docs/AI_SYSTEM.md`
- [x] Write `docs/SECURITY.md`
- [x] Write `docs/DEVELOPMENT.md`
- [x] Write `docs/DECISIONS.md` (ADR-001..ADR-007)
- [x] Write `tasks/TODO.md` (this file)
- [x] Write `tasks/CURRENT_TASK.md` (no active task)
- [x] Write `README.md`
- [x] Write `.gitignore`
- [x] Write `.env.example` (placeholders only, no secrets)
- [x] Cross-reference documentation files
- [x] Verify no product functionality implemented; no secrets committed

---

## PHASE 1 — Project Foundation

- [ ] Add `docker-compose.yml` for local PostgreSQL (with pgvector) and Redis
- [ ] Scaffold `backend/` FastAPI application (project layout per `docs/ARCHITECTURE.md`)
- [ ] Add typed settings/config module (loads from env; one settings object)
- [ ] Add structured JSON logging with `request_id` correlation
- [ ] Add `GET /api/v1/health` endpoint (checks process, DB, Redis)
- [ ] Add backend test for the health endpoint
- [ ] Set up `ruff` + `black` + `mypy`/`pyright` for backend; wire into a make/script target
- [ ] Scaffold `frontend/` Next.js app (App Router, TypeScript, Tailwind)
- [ ] Configure ESLint + Prettier + `strict` TypeScript for frontend
- [ ] Add a frontend page that calls `/api/v1/health` and renders status
- [ ] Add a frontend test for the health page
- [ ] Add pre-commit hooks (lint, format, type-check)
- [ ] Add CI workflow (install, lint, type-check, test for backend and frontend)
- [ ] Add `scripts/` entries for dev up/down, test, lint
- [ ] Update `README.md` with real local-setup steps
- [ ] Update `PROJECT_MEMORY.md` (Completed Work, Current Phase)

---

## PHASE 2 — Authentication & Multi-Tenancy

- [ ] Add Alembic and the initial migration setup
- [ ] Create `organizations`, `users`, `memberships` tables (migration)
- [ ] Implement password hashing (argon2/bcrypt) or magic-link auth (decide + ADR)
- [ ] Implement signup (creates user + organization + owner membership)
- [ ] Implement login / logout (signed HTTP-only secure session cookie)
- [ ] Implement `current_user` FastAPI dependency (resolves user + memberships)
- [ ] Implement `require_role` dependency (owner/admin/member)
- [ ] Implement tenancy helper: scoped DB session / query guard by `organization_id`
- [ ] Add `/api/v1/auth/*` and `/api/v1/me` endpoints
- [ ] Frontend: login page, auth context, protected route wrapper
- [ ] Tests: auth flows, role checks, tenancy scoping (cross-tenant denial)
- [ ] Add audit-log table + helper; record login events
- [ ] Update `docs/SECURITY.md` / `docs/DECISIONS.md` if choices differ from plan
- [ ] Update `PROJECT_MEMORY.md`

---

## PHASE 3 — PostgreSQL Data Layer

- [ ] Establish SQLAlchemy base, session management, and migration conventions
- [ ] Create `connections` table (encrypted token columns, status, scopes, metadata)
- [ ] Implement app-level envelope encryption for token fields (key from env/KMS)
- [ ] Create `sync_jobs` table (job_type, resource, status, cursor, stats, error)
- [ ] Create `source_objects` table (unified retrievable/citable projection)
- [ ] Add base repository pattern with mandatory `organization_id` filtering
- [ ] Add DB indexes for tenant + lookup columns
- [ ] Tests: encryption round-trip, repository tenancy enforcement, migrations up/down
- [ ] Update `docs/DATA_MODEL.md` with the finalized columns for these tables
- [ ] Update `PROJECT_MEMORY.md`

---

## PHASE 4 — Slack Connector

- [ ] Verify current Slack OAuth v2 + Web API docs (scopes, endpoints, rate limits)
- [ ] Implement shared connector base (pagination, rate-limit handling, backoff, mappers)
- [ ] Implement Slack OAuth: authorize redirect + code exchange + token storage
- [ ] Implement Slack disconnect (delete/revoke token, mark connection)
- [ ] Create Slack tables: `slack_workspaces`, `slack_users`, `slack_channels`,
      `slack_messages`, `slack_threads` (migration)
- [ ] Implement fetchers: team info, users, channels, channel history, thread replies
- [ ] Implement normalization: Slack records → normalized rows + `source_objects`
- [ ] Implement initial backfill (bounded time window per channel)
- [ ] Implement incremental sync (per-channel cursor / last ts)
- [ ] Frontend: "Connect Slack" flow + connection status view
- [ ] Tests: OAuth handler, fetch pagination (fixtures), normalization, incremental cursor
- [ ] Update `docs/INTEGRATIONS.md` with verified Slack details
- [ ] Update `PROJECT_MEMORY.md`

---

## PHASE 5 — Background Job System

- [ ] Choose worker library (RQ / Arq / Celery) and record ADR
- [ ] Add worker process + Redis broker configuration
- [ ] Define job interface: idempotent, resumable, records state in `sync_jobs`
- [ ] Move Slack initial + incremental sync onto the queue
- [ ] Add scheduled/periodic trigger for incremental syncs
- [ ] Add job retry + dead-letter handling + failure surfacing on the connection
- [ ] Add job observability (status endpoint, structured logs with `job_id`)
- [ ] Tests: job enqueue/execute, retry, resume from cursor, failure state
- [ ] Update `docs/ARCHITECTURE.md` (Background Processing) and `PROJECT_MEMORY.md`

---

## PHASE 6 — Jira Connector

- [ ] Verify current Atlassian OAuth 2.0 (3LO) + Jira Cloud REST docs (JQL, changelog,
      pagination, permissions)
- [ ] Implement Jira OAuth: authorize + exchange + cloudId resolution + token storage
- [ ] Implement Jira disconnect
- [ ] Create Jira tables: `jira_projects`, `jira_users`, `jira_issues`, `jira_comments`,
      `jira_issue_history` (migration)
- [ ] Implement fetchers: projects, users, issues (JQL), comments, changelog
- [ ] Implement normalization → normalized rows + `source_objects`
- [ ] Implement initial backfill (per project, bounded by updated date)
- [ ] Implement incremental sync (`updated >= last_sync` via JQL)
- [ ] Run Jira sync via the job system
- [ ] Frontend: "Connect Jira" flow + status
- [ ] Tests: OAuth, fetch (fixtures), normalization, incremental
- [ ] Update `docs/INTEGRATIONS.md` with verified Jira details
- [ ] Update `PROJECT_MEMORY.md`

---

## PHASE 7 — Unified Data Model

- [ ] Finalize `source_objects` schema across Slack + Jira (object_type taxonomy)
- [ ] Backfill `source_objects` for all existing synced records
- [ ] Add a normalization contract/tests every connector must satisfy
- [ ] Add `documents`/`source_objects` read API for retrieval layers
- [ ] Add PostgreSQL full-text (`tsvector` column + GIN index) on `source_objects.text`
- [ ] Tests: taxonomy coverage, backfill correctness, FTS returns expected rows
- [ ] Update `docs/DATA_MODEL.md` and `PROJECT_MEMORY.md`

---

## PHASE 8 — Entity Resolution

- [ ] Create `people` table (canonical person) + links to `slack_users`, `jira_users`
- [ ] Implement matching: email match, exact name match, manual override support
- [ ] Populate `author_person_id` on `source_objects`
- [ ] Add a review/merge/split API for ambiguous matches
- [ ] Add `search_by_person` support data (person → records index)
- [ ] Tests: match precision/recall on fixtures, merge/split, no cross-tenant merge
- [ ] Update `docs/DATA_MODEL.md`, `docs/AI_SYSTEM.md`, `PROJECT_MEMORY.md`

---

## PHASE 9 — Search Engine

- [ ] Implement keyword search over `source_objects` (FTS) with metadata filters
- [ ] Implement filters: tenant, system, object_type, time window, container, person
- [ ] Implement result model (score, snippet, source ref, metadata)
- [ ] Add `/api/v1/search` (internal/dev) endpoint
- [ ] Add pagination + result limits
- [ ] Tests: relevance on seeded data, filter correctness, tenancy enforcement
- [ ] Update `docs/AI_SYSTEM.md` and `PROJECT_MEMORY.md`

---

## PHASE 10 — Semantic Search / RAG

- [ ] Choose embedding model/provider; record ADR (data-use terms reviewed)
- [ ] Enable `pgvector`; create `embeddings` table with metadata + vector column + index
- [ ] Implement per-source chunking (Slack thread-aware; Jira issue/comment; …)
- [ ] Implement embedding job (batch, rate-limited, records model + token_count)
- [ ] Implement re-embedding job (model/chunking change)
- [ ] Implement vector similarity search with tenant + metadata pre-filter
- [ ] Tests: chunk boundaries, embedding job idempotency, vector recall on fixtures,
      pre-filter applied before similarity
- [ ] Update `docs/AI_SYSTEM.md`, `docs/DATA_MODEL.md`, `docs/DECISIONS.md`,
      `PROJECT_MEMORY.md`

---

## PHASE 11 — Hybrid Retrieval

- [ ] Implement parallel keyword + vector retrieval
- [ ] Implement fusion (reciprocal rank fusion / weighted score)
- [ ] Choose and implement reranking (hosted reranker / LLM / fusion-only); record ADR
- [ ] Implement query understanding (intent, entities, time window, systems)
- [ ] Produce a single ranked candidate list feeding the evidence engine
- [ ] Tests: fusion correctness, rerank improves ordering on fixtures, query-understanding
      extraction
- [ ] Update `docs/AI_SYSTEM.md` and `PROJECT_MEMORY.md`

---

## PHASE 12 — Evidence Engine

- [ ] Create `evidence` table (source ref, quoted span, author, time, reason, level)
- [ ] Implement conversion: ranked candidate → evidence item with provenance
- [ ] Implement evidence level assignment (Confirmed/Likely/Possible/Unknown) rules
- [ ] Implement `get_evidence` retrieval by id
- [ ] Enforce evidence immutability within an investigation
- [ ] Tests: provenance completeness, level rules, immutability, tenancy
- [ ] Update `docs/AI_SYSTEM.md`, `docs/DATA_MODEL.md`, `PROJECT_MEMORY.md`

---

## PHASE 13 — GitHub Connector

- [ ] Decide OAuth App vs GitHub App; record ADR; verify current GitHub API docs
- [ ] Implement GitHub auth (flow per the decision) + token/installation storage
- [ ] Implement GitHub disconnect
- [ ] Create GitHub tables: `github_organizations`, `github_users`,
      `github_repositories`, `github_issues`, `github_pull_requests`, `github_reviews`,
      `github_comments`, `github_commits` (migration)
- [ ] Implement fetchers: org, repos, issues, PRs, reviews, comments, commits
- [ ] Implement normalization → normalized rows + `source_objects`
- [ ] Implement initial backfill (bounded) + incremental sync
- [ ] Run GitHub sync via the job system
- [ ] Extend entity resolution to `github_users`
- [ ] Frontend: "Connect GitHub" flow + status
- [ ] Tests: auth, fetch (fixtures), normalization, incremental, entity resolution
- [ ] Update `docs/INTEGRATIONS.md` with verified GitHub details
- [ ] Update `PROJECT_MEMORY.md`

---

## PHASE 14 — Cross-System Relationships

- [ ] Create `relationships` table (from/to type+id, relation, confidence, evidence ref)
- [ ] Implement extractors: Jira keys in text, GitHub `closes/fixes #N`, pasted URLs
- [ ] Implement same-person links from entity resolution
- [ ] Implement time-window co-occurrence linking
- [ ] Implement `find_related_entities(record)` lookup
- [ ] Add a relationship-build job (runs after sync)
- [ ] Tests: extractor precision on fixtures, `find_related_entities` correctness, tenancy
- [ ] Update `docs/DATA_MODEL.md`, `docs/INTEGRATIONS.md`, `PROJECT_MEMORY.md`

---

## PHASE 15 — Investigation Engine

- [ ] Choose generation LLM provider/model; record ADR (data-use terms reviewed)
- [ ] Create `investigations` and `investigation_events` tables
- [ ] Implement the deterministic workflow skeleton (stages, state machine)
- [ ] Implement investigation lifecycle: create, run (as a job), status, cancel
- [ ] Implement per-investigation bounds (iterations, evidence count, tokens, wall-clock)
- [ ] Implement `investigation_events` trace writing for every stage
- [ ] Add `/api/v1/investigations` CRUD + `/events` stream endpoint
- [ ] Tests: state machine transitions, bounds enforced, trace completeness, tenancy
- [ ] Update `docs/AI_SYSTEM.md`, `docs/ARCHITECTURE.md`, `docs/DECISIONS.md`,
      `PROJECT_MEMORY.md`

---

## PHASE 16 — Agent Tools

- [ ] Define tool schema + dispatcher (each tool takes `investigation_id`, runs as user)
- [ ] Implement `search_slack`, `search_jira`, `search_github`
- [ ] Implement `get_slack_thread`, `get_jira_issue`, `get_github_pr`, `get_commit`
- [ ] Implement `find_related_entities`, `search_by_person`, `search_by_time`
- [ ] Implement `get_evidence`
- [ ] Enforce: every tool returns only tenant + permission authorized data
- [ ] Record each tool call as an `investigation_events` entry
- [ ] Implement the bounded tool-use loop within a stage
- [ ] Tests: each tool's output shape, authorization filtering, loop bound
- [ ] Update `docs/AI_SYSTEM.md` and `PROJECT_MEMORY.md`

---

## PHASE 17 — Investigation Planning

- [ ] Implement the plan stage: question → structured plan (targets, time window,
      entities, stages to run)
- [ ] Persist the plan on the investigation
- [ ] Feed the plan into the gather stage to scope tool calls
- [ ] Handle under-specified questions (ask for / infer a time window)
- [ ] Tests: plan structure on fixtures, plan constrains gathering, degenerate inputs
- [ ] Update `docs/AI_SYSTEM.md` and `PROJECT_MEMORY.md`

---

## PHASE 18 — Timeline Engine

- [ ] Implement event extraction from evidence (timestamped occurrences)
- [ ] Implement ordering + de-duplication across systems
- [ ] Implement phase tagging (first alert / diagnosis / change / resolution)
- [ ] Attach source links + evidence ids to each timeline entry
- [ ] Tests: ordering correctness, dedup, phase tagging on a seeded incident
- [ ] Update `docs/AI_SYSTEM.md`, `docs/PRODUCT.md`, `PROJECT_MEMORY.md`

---

## PHASE 19 — Reasoning & Root Cause

- [ ] Implement hypothesis generation (candidate causes, each tied to evidence)
- [ ] Implement evidence weighing (supporting vs contradictory per hypothesis)
- [ ] Implement confidence derivation (evidence levels + contradictions → overall)
- [ ] Implement selection of the best-supported root cause (or "insufficient evidence")
- [ ] Implement unresolved-questions extraction
- [ ] Tests: correct root cause on seeded scenarios, contradictions surfaced, low-evidence
      case returns "insufficient"
- [ ] Update `docs/AI_SYSTEM.md`, `docs/PRODUCT.md`, `PROJECT_MEMORY.md`

---

## PHASE 20 — Citation / Source System

- [ ] Implement citation model: claims/timeline/people reference evidence ids
- [ ] Implement the report assembler (structured JSON + prose) with all PRODUCT.md sections
- [ ] Implement uncited-claim detection: flag/reject conclusions with no evidence link
- [ ] Implement deep source links back to Slack/Jira/GitHub records
- [ ] Handle broken links for deleted source data
- [ ] Tests: every conclusion cited, assembler output schema, link resolution
- [ ] Update `docs/PRODUCT.md`, `docs/AI_SYSTEM.md`, `PROJECT_MEMORY.md`

---

## PHASE 21 — Permissions

- [ ] Implement per-source authorization computation (channels/projects/repos a user may
      see); verify each source's permission model against official docs
- [ ] Implement the permission-aware retrieval pre-filter (applied before rerank + LLM)
- [ ] Wire the filter into search, hybrid retrieval, evidence engine, and every agent tool
- [ ] Conservative default: deny when authorization is unknown
- [ ] Add tests asserting the LLM context never contains unauthorized evidence
- [ ] Add tests for each retrieval path: user A cannot see user B's restricted records
- [ ] Update `docs/SECURITY.md`, `docs/AI_SYSTEM.md`, `docs/DECISIONS.md` (RLS decision),
      `PROJECT_MEMORY.md`

---

## PHASE 22 — Investigation UI

- [ ] Investigation list page
- [ ] New investigation form (question, optional time window / systems / people)
- [ ] Investigation detail: streamed progress from `investigation_events`
- [ ] Timeline view with source links
- [ ] Report view: all sections, expandable evidence, confidence, contradictions,
      unresolved questions
- [ ] Click-through from any conclusion to its evidence and to the source record
- [ ] Empty/error/loading states; cancel a running investigation
- [ ] Frontend tests: form, streaming render, report render, evidence expansion
- [ ] Update `docs/PRODUCT.md` and `PROJECT_MEMORY.md`

---

## PHASE 23 — Hallucination & Quality Controls

- [ ] Enforce source-grounding system prompt (no outside knowledge for factual claims)
- [ ] Enforce citation requirement in the assembler (hard fail on uncited root cause)
- [ ] Implement contradiction detection surfacing
- [ ] Enforce maximum agent iterations at the engine level
- [ ] Add response validation (schema + citation validity + level consistency)
- [ ] Add a "confidence too low" path that returns questions instead of a forced answer
- [ ] Tests: injected-unsupported-claim is caught, contradiction surfaced, iteration cap
- [ ] Update `docs/AI_SYSTEM.md` and `PROJECT_MEMORY.md`

---

## PHASE 24 — Observability

- [ ] Structured logs across API → worker → LLM with correlation ids
- [ ] Metrics: sync throughput/lag, search latency, investigation duration, tool-call
      counts, token usage
- [ ] Tracing across the investigation stages
- [ ] Per-investigation cost record (tokens in/out × price, tool calls)
- [ ] Health/readiness endpoints for all processes
- [ ] Error tracking integration
- [ ] Tests: correlation ids propagate, cost record populated
- [ ] Update `docs/ARCHITECTURE.md` (Observability) and `PROJECT_MEMORY.md`

---

## PHASE 25 — Security

- [ ] Security review of OAuth token storage + encryption + key handling
- [ ] Verify logging allowlist (no tokens/secrets/full bodies) across the codebase
- [ ] Audit-log coverage: connections, syncs, investigations, evidence access
- [ ] Prompt-injection handling review (source content treated as untrusted data)
- [ ] Rate limiting on auth and investigation endpoints
- [ ] Dependency vulnerability scan in CI
- [ ] Data deletion: tenant offboarding + per-connection purge jobs, verified
- [ ] Tests: cross-tenant isolation suite, deletion verification, audit completeness
- [ ] Run `/security-review` on the branch
- [ ] Update `docs/SECURITY.md` and `PROJECT_MEMORY.md`

---

## PHASE 26 — Evaluation Dataset

- [ ] Build 5–10 seeded investigation scenarios (Slack+Jira+GitHub fixtures, known answer)
- [ ] Define scoring: root-cause correctness, key-evidence recall, citation validity,
      permission compliance, timeline accuracy
- [ ] Implement an eval runner that executes scenarios and scores output
- [ ] Add a permission-compliance scenario (unauthorized evidence must never appear)
- [ ] Wire eval into CI (report, not necessarily gating at first)
- [ ] Document baseline scores
- [ ] Update `docs/AI_SYSTEM.md` and `PROJECT_MEMORY.md`

---

## PHASE 27 — Cost Controls

- [ ] Enforce per-investigation token + tool-call + wall-clock budgets (hard stop)
- [ ] Add per-tenant daily/monthly investigation + spend limits
- [ ] Implement prompt caching for the system prompt + tool schemas
- [ ] Add batching / rate limiting for embedding jobs
- [ ] Add cost dashboards / alerts
- [ ] Surface "investigation stopped: budget reached" cleanly in the UI
- [ ] Tests: budget stop triggers, tenant limit enforced
- [ ] Update `docs/AI_SYSTEM.md`, `docs/ARCHITECTURE.md`, `PROJECT_MEMORY.md`

---

## PHASE 28 — V1 Production Deployment

- [ ] Choose hosting (container platform, managed PostgreSQL+pgvector, managed Redis)
- [ ] Production configuration + secrets management (KMS/secrets manager)
- [ ] Database migration strategy for deploys
- [ ] CI/CD pipeline: build, test, migrate, deploy
- [ ] TLS, custom domain, security headers
- [ ] Backups + restore runbook for PostgreSQL
- [ ] Monitoring + alerting in production
- [ ] Load/smoke test with realistic data volume
- [ ] Deployment runbook in `docs/` or `scripts/`
- [ ] Update `docs/ARCHITECTURE.md` (Deployment), `docs/DECISIONS.md`, `PROJECT_MEMORY.md`

---

## PHASE 29 — V1 Beta

- [ ] Onboard 1–3 friendly tenants
- [ ] Collect investigation feedback (correctness, usefulness, missing evidence)
- [ ] Fix top issues from beta feedback
- [ ] Tune retrieval / reranking / prompts against real data
- [ ] Verify cost per investigation is within target
- [ ] Confirm permission compliance with real tenant data
- [ ] Update `PROJECT_MEMORY.md` (Known Problems, Open Questions) from findings

---

## PHASE 30 — V1 Definition of Done

- [ ] Slack, Jira, GitHub connectors: OAuth, initial + incremental sync, tested
- [ ] Unified data model + entity resolution + cross-system relationships working
- [ ] Hybrid retrieval (keyword + semantic + rerank) with permission pre-filter
- [ ] Evidence engine with provenance and levels
- [ ] Deterministic investigation workflow producing all `docs/PRODUCT.md` report sections
- [ ] Every conclusion in every report is cited to source evidence
- [ ] LLM never receives unauthorized evidence (verified by tests + eval scenario)
- [ ] Per-investigation and per-tenant cost controls enforced
- [ ] Observability: logs, metrics, tracing, per-investigation cost
- [ ] Security: encrypted tokens, audit logs, cross-tenant isolation suite, deletion,
      `/security-review` clean
- [ ] Evaluation dataset passing at the documented baseline
- [ ] Deployed to production with backups, monitoring, and a runbook
- [ ] `PROJECT_MEMORY.md` current state updated from "initialization" to "V1 shipped"
