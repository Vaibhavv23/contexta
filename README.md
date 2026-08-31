# Enterprise Context & Investigation Agent

## What is this?

An AI system that connects fragmented company data across **Slack, Jira, and GitHub** and
reconstructs what happened around a business or engineering problem — with every
conclusion traceable to its source.

> "Give me a business/engineering problem. I'll investigate the company's fragmented
> data, connect the evidence, reconstruct what happened, explain why, and show you
> exactly where every conclusion came from."

## Product Vision

Engineering and operational context is scattered across chat, issue trackers, and code
hosting. Reconstructing an incident or a decision is slow, manual, and rarely backed by
verifiable sources. This product does that reconstruction automatically: it retrieves
relevant records from connected systems, links them into a timeline and narrative, forms
evidence-backed hypotheses about root cause, and presents a cited report.

See [PROJECT_MEMORY.md](./PROJECT_MEMORY.md) and [docs/PRODUCT.md](./docs/PRODUCT.md).

## V1

- Sources: Slack, Jira, GitHub (read-only, one workspace/org each).
- Multi-tenant organizations with permission-aware retrieval.
- Keyword + semantic (pgvector) + hybrid search over synced records.
- Evidence engine with full provenance and confidence levels.
- Deterministic investigation workflow: plan → gather → cross-reference → timeline →
  hypotheses → reasoning → cited report.
- Investigation UI.

Explicit non-goals: generic enterprise search, write-back to source systems, sources
beyond the three above, fully autonomous agent behavior, self-hosted LLMs. See
[PROJECT_MEMORY.md](./PROJECT_MEMORY.md) §10.

## Architecture

Modular monolith: Next.js frontend → FastAPI backend → PostgreSQL (+ pgvector) + Redis +
hosted LLM APIs, with read-only connectors for Slack, Jira, and GitHub and an
investigation engine orchestrating retrieval, evidence, timeline, and reasoning.

See [docs/ARCHITECTURE.md](./docs/ARCHITECTURE.md) and
[docs/DECISIONS.md](./docs/DECISIONS.md).

## Repository Structure

```text
CLAUDE.md              Instructions for Claude Code sessions
PROJECT_MEMORY.md      High-level persistent project memory
README.md              This file
.env.example           Environment variable placeholders
frontend/              Next.js app (not yet scaffolded)
backend/               FastAPI app + worker (not yet scaffolded)
docs/                  Product, architecture, data model, integrations, AI, security,
                       development, and decision docs
tasks/                 TODO.md roadmap, CURRENT_TASK.md, completed/ archive
tests/                 Cross-cutting / integration tests (not yet populated)
scripts/               Dev and ops scripts (not yet populated)
```

## Development Workflow

One small task at a time, from [tasks/TODO.md](./tasks/TODO.md):

```text
Task → Inspect → Plan → Implement → Test → Review → Document → Commit
```

Full standards and workflow: [docs/DEVELOPMENT.md](./docs/DEVELOPMENT.md).

## Documentation

| Doc | Purpose |
| --- | --- |
| [PROJECT_MEMORY.md](./PROJECT_MEMORY.md) | Project memory: vision, scope, roadmap, current state |
| [docs/PRODUCT.md](./docs/PRODUCT.md) | V1 product definition and output shape |
| [docs/ARCHITECTURE.md](./docs/ARCHITECTURE.md) | Planned system architecture |
| [docs/DATA_MODEL.md](./docs/DATA_MODEL.md) | Planned entities / schema specification |
| [docs/INTEGRATIONS.md](./docs/INTEGRATIONS.md) | Slack / Jira / GitHub integration strategy |
| [docs/AI_SYSTEM.md](./docs/AI_SYSTEM.md) | Retrieval, RAG, investigation agent design |
| [docs/SECURITY.md](./docs/SECURITY.md) | Authn/authz, multi-tenancy, LLM data boundaries |
| [docs/DEVELOPMENT.md](./docs/DEVELOPMENT.md) | Dev workflow, coding standards, stack |
| [docs/DECISIONS.md](./docs/DECISIONS.md) | Architecture Decision Records |
| [tasks/TODO.md](./tasks/TODO.md) | Complete V1 implementation roadmap (30 phases) |
| [tasks/CURRENT_TASK.md](./tasks/CURRENT_TASK.md) | The single task currently in progress |

## Current Status

```text
Status: Project initialization
```

Documentation and the memory system are in place. No product functionality — frontend,
backend, database, authentication, integrations, search, RAG, agent, investigation
engine, or deployment — has been implemented yet. Implementation begins with
[tasks/TODO.md](./tasks/TODO.md) Phase 1.
