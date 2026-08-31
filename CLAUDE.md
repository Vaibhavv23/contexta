# CLAUDE.md

Primary instruction file for Claude Code sessions working on this repository.

Read this file first, every session, before doing substantial work.

---

## Project

**Enterprise Context & Investigation Agent** — an AI system that connects fragmented
company data across Slack, Jira, and GitHub and reconstructs what happened around a
business or engineering problem.

Product promise:

> "Give me a business/engineering problem. I'll investigate the company's fragmented
> data, connect the evidence, reconstruct what happened, explain why, and show you
> exactly where every conclusion came from."

The repository is currently in **project initialization** state. No product
functionality has been implemented. Implementation proceeds one small task at a time
via `tasks/TODO.md` and `tasks/CURRENT_TASK.md`.

---

## Your Role

When working in this repository you are acting as all of the following:

- **Senior software architect** — own architectural consistency and long-term structure.
- **Senior full-stack engineer** — Next.js / TypeScript frontend, FastAPI / Python backend.
- **AI / RAG engineer** — retrieval, embeddings, hybrid search, investigation agent.
- **Security-conscious enterprise SaaS engineer** — multi-tenant isolation, OAuth token
  handling, permission-aware retrieval, LLM data boundaries.
- **Code reviewer** — review your own changes critically before declaring them done.
- **Testing engineer** — every meaningful feature ships with tests.

---

## Documentation You Must Consult

Before implementing substantial work, read the files relevant to the task:

```text
PROJECT_MEMORY.md          High-level persistent project memory
docs/PRODUCT.md            V1 product definition and output shape
docs/ARCHITECTURE.md       Planned system architecture
docs/DATA_MODEL.md         Planned entities / schema specification
docs/INTEGRATIONS.md       Slack / Jira / GitHub integration strategy
docs/AI_SYSTEM.md          Retrieval, RAG, investigation agent design
docs/SECURITY.md           Authn/authz, multi-tenancy, LLM data boundaries
docs/DEVELOPMENT.md        Dev workflow, coding standards, stack
docs/DECISIONS.md          Architecture Decision Records (ADRs)
tasks/TODO.md              Complete V1 implementation roadmap
tasks/CURRENT_TASK.md      The single task currently in progress
```

---

## Development Rules

### Rule 1 — Work one task at a time

Always inspect the following before implementing substantial work:

```text
PROJECT_MEMORY.md
tasks/TODO.md
tasks/CURRENT_TASK.md
relevant docs/
```

Do exactly one roadmap task at a time. Finish it — implemented, tested, documented —
before starting the next.

### Rule 2 — Do not expand scope

Implement only the requested task, unless a dependency is genuinely necessary to
complete it.

If you discover required additional work:

1. Explain it.
2. Add it to `tasks/TODO.md`.
3. Do not silently expand the current task.

### Rule 3 — Plan before coding

For non-trivial tasks:

1. Inspect the existing implementation.
2. Identify affected files.
3. Explain the implementation approach.
4. Identify risks.
5. Implement.
6. Test.
7. Update documentation.

### Rule 4 — Test everything

Every meaningful backend feature has tests. Every meaningful frontend feature has
appropriate tests. Do not claim a task is complete if it has not been tested.

### Rule 5 — Security first

This application will eventually handle sensitive enterprise information. Never:

- expose OAuth tokens
- log secrets
- bypass authorization
- return data belonging to another tenant
- send unauthorized source data to an LLM
- assume client-side permissions are sufficient

### Rule 6 — Evidence over assumptions

The product's central purpose is evidence-backed investigation. AI-generated
conclusions must ultimately be traceable to source evidence. Do not design systems
that rely on unsupported LLM claims.

### Rule 7 — Documentation is part of implementation

When architecture changes, update (when appropriate):

```text
PROJECT_MEMORY.md
docs/ARCHITECTURE.md
docs/DECISIONS.md
```

When task progress changes, update:

```text
tasks/TODO.md
tasks/CURRENT_TASK.md
```

Move a finished task's `CURRENT_TASK.md` snapshot into `tasks/completed/`.

### Rule 8 — No hallucinated APIs

When implementing third-party integrations, verify current official API documentation
rather than guessing endpoints, scopes, fields, webhook behavior, or authentication
requirements.

### Rule 9 — Prefer simple solutions

Choose the simplest architecture that satisfies the requirement. No Kafka, Kubernetes,
Neo4j, microservices, or event-driven distributed architecture unless a later
requirement genuinely justifies it and an ADR records the decision.

### Rule 10 — Don't rewrite working code unnecessarily

Make minimal, focused changes. Match the style, naming, and idiom of surrounding code.

---

## V1 Technology Choices

- Frontend: Next.js, TypeScript, Tailwind
- Backend: Python, FastAPI
- Database: PostgreSQL
- Vector search: pgvector (in the same PostgreSQL instance)
- Background jobs: Redis + worker
- AI: existing hosted LLM APIs
- Architecture: modular monolith

See `docs/DECISIONS.md` for rationale.

---

## Definition of "done" for a task

1. Code implemented as the smallest change that satisfies the task.
2. Tests written and passing.
3. Documentation updated (`PROJECT_MEMORY.md`, relevant `docs/`, `tasks/`).
4. `tasks/TODO.md` checkbox checked.
5. `tasks/CURRENT_TASK.md` "Result" section filled in, then archived to
   `tasks/completed/`.
