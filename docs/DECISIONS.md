# Architecture Decisions

Architecture Decision Records (ADRs) for the Enterprise Context & Investigation Agent.

Format per ADR: Status, Decision, Reason, Alternatives considered, Why rejected, Date.
Statuses: Proposed / Accepted / Superseded / **Marked for future review**.

An ADR marked for future review is a working assumption, not an immutable commitment.
Revisit it when the relevant phase begins or when a requirement changes.

---

## ADR-001 — Initial Architecture

**Status:** Accepted

**Decision:** Use a modular monolith for V1 — one backend codebase and deployable
(plus a worker sharing that codebase), with clear internal module boundaries.

**Reason:** Simpler development, testing, and deployment for a solo / small team. Lets us
move task-by-task without cross-service coordination.

**Alternatives considered:** Microservices.

**Why rejected:** Unnecessary operational complexity (service discovery, deployment
orchestration, distributed tracing, network failure modes) for V1's scale and team size.

**Date:** 2026-08-31

---

## ADR-002 — PostgreSQL as the Primary Database

**Status:** Accepted

**Decision:** PostgreSQL is the single system of record for all application and synced
source data.

**Reason:** Mature, well-understood, transactional, supports JSONB for raw payloads,
full-text search for keyword retrieval, and pgvector for embeddings — covering V1's needs
in one system.

**Alternatives considered:** PostgreSQL + a separate document store; MySQL.

**Why rejected:** A second datastore adds operational overhead with no V1 benefit;
PostgreSQL's JSONB covers the semi-structured needs.

**Date:** 2026-08-31

---

## ADR-003 — pgvector for Vector Search

**Status:** Accepted · Marked for future review (revisit if scale/recall demands it)

**Decision:** Use the `pgvector` extension in the same PostgreSQL instance for embedding
storage and similarity search in V1.

**Reason:** No extra infrastructure; transactional consistency with the source rows;
metadata pre-filtering (tenant, permissions, time) via ordinary SQL; sufficient for V1
data volumes.

**Alternatives considered:** Dedicated vector database (Pinecone, Weaviate, Qdrant,
Milvus).

**Why rejected:** Operational and cost overhead, another system to secure and make
multi-tenant, and data-sync complexity — not justified at V1 scale.

**Review trigger:** vector row counts or latency exceed what a single Postgres instance
serves comfortably, or advanced ANN features are needed.

**Date:** 2026-08-31

---

## ADR-004 — Modular Monolith over Microservices for V1

**Status:** Accepted

**Decision:** Keep all backend modules (auth, connectors, sync, search, evidence,
investigation, agent, permissions) in one codebase with enforced internal boundaries;
no network-separated services.

**Reason:** Same as ADR-001 — this ADR records it explicitly as a standing constraint so
future tasks do not drift toward service extraction without a new ADR.

**Alternatives considered:** Extract connectors or the investigation engine as separate
services.

**Why rejected:** Premature. Module boundaries in code give most of the benefit with
none of the operational cost.

**Note on multi-tenancy isolation:** V1 uses shared-schema, application-level
`organization_id` scoping. Whether to add PostgreSQL row-level security as
defense-in-depth is an open question (see PROJECT_MEMORY §30) — decide before Phase 21,
record as its own ADR.

**Date:** 2026-08-31

---

## ADR-005 — Deterministic Investigation Workflow Before Autonomous Agent

**Status:** Accepted

**Decision:** V1's investigation is a deterministic, staged workflow (plan → gather →
cross-reference → timeline → hypotheses → reasoning → report). The LLM operates within a
stage and within a bounded toolset; it does not choose the overall procedure or run
open-ended.

**Reason:** Predictability, testability, bounded cost, and easier evidence/citation
guarantees. An autonomous agent is harder to evaluate and to keep within permission and
cost boundaries.

**Alternatives considered:** A fully autonomous tool-using agent with a single objective
prompt.

**Why rejected:** Too hard to make reliable, auditable, and cost-bounded for V1. Can be
revisited once the deterministic version is solid and the evaluation dataset exists.

**Date:** 2026-08-31

---

## ADR-006 — Slack is the First Integration

**Status:** Accepted

**Decision:** Implement the Slack connector before Jira and GitHub.

**Reason:** Slack holds the discussion narrative around incidents (who noticed, what was
tried, when it resolved) — the connective tissue an investigation needs — and exercises
the OAuth, sync, chunking, and permission machinery the other connectors will reuse.

**Alternatives considered:** GitHub first (structured, well-documented API).

**Why rejected:** GitHub data alone underdelivers on the "reconstruct what happened"
promise without the discussion context; Slack forces the harder design questions early.

**Date:** 2026-08-31

---

## ADR-007 — V1 Focuses on Engineering Investigation

**Status:** Accepted

**Decision:** V1 targets engineering / production-incident investigation, not generic
enterprise search across all company data.

**Reason:** A narrow, deep product that reliably answers "why did this break and what
changed" is more valuable and more achievable than a shallow universal search tool.
Keeps scope, data sources, and evaluation tractable.

**Alternatives considered:** Position as a general enterprise knowledge/search assistant.

**Why rejected:** Broad scope dilutes quality, multiplies integrations, and makes
evidence/permission guarantees much harder. Broader scope is future vision, not V1.

**Date:** 2026-08-31

---

## Template for New ADRs

```text
## ADR-NNN — <title>

**Status:** Proposed | Accepted | Superseded by ADR-XXX | Marked for future review

**Decision:** <what we will do>

**Reason:** <why>

**Alternatives considered:** <options>

**Why rejected:** <why not>

**Date:** YYYY-MM-DD
```
