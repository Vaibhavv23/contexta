# Project Memory

High-level persistent memory for the Enterprise Context & Investigation Agent.
Update this file whenever project direction, architecture, scope, or task status changes.

---

## 1. Project Name

Enterprise Context & Investigation Agent

(Repository directory: `contexta`.)

## 2. One-Line Description

An AI agent that investigates fragmented company data across Slack, Jira, and GitHub to
reconstruct what happened around a business or engineering problem, explain why, and cite
every piece of supporting evidence.

## 3. Product Vision

Engineering and operational knowledge is scattered across chat, issue trackers, and code
hosting. When something breaks or a decision needs context, a human spends hours manually
stitching together Slack threads, Jira tickets, PRs, and commits. This product does that
stitching automatically: given a problem statement, it retrieves relevant records from
connected systems, links them into a coherent narrative and timeline, forms
evidence-backed hypotheses about root cause, and presents conclusions that are always
traceable to their sources.

## 4. Core Problem

Company context is fragmented across many SaaS tools. No single system knows the whole
story of an incident or decision. Reconstruction is manual, slow, and error-prone, and
the resulting explanations are rarely backed by verifiable sources.

## 5. Problem Statement

Given a business or engineering problem, a user cannot quickly and reliably answer:
what happened, in what order, why, who was involved, which change was responsible, and
what evidence supports each of those conclusions — because the relevant data lives in
separate systems with separate search, separate permissions, and no shared model.

## 6. Product Promise

> "Give me a business/engineering problem. I'll investigate the company's fragmented
> data, connect the evidence, reconstruct what happened, explain why, and show you
> exactly where every conclusion came from."

## 7. Target Users

- Engineers and engineering managers doing incident retrospectives and root-cause analysis.
- On-call responders needing fast context on an active problem.
- Tech leads and staff engineers reconstructing the history of a system or decision.
- Support and operations staff who need to know what engineering did and when.

Primary V1 persona: an engineer investigating a production incident.

## 8. Primary Use Case

Post-incident and mid-incident investigation of an engineering problem: "Why did checkout
fail yesterday?" — where the answer requires correlating Slack discussion, Jira tickets,
and GitHub PRs/commits around a time window.

## 9. V1 Scope

- Organization / user / membership model with multi-tenant isolation.
- OAuth connection to Slack, Jira, and GitHub (one workspace/org each per connection).
- Sync of messages/threads (Slack), issues/comments/history (Jira), and
  issues/PRs/reviews/comments/commits (GitHub) into PostgreSQL.
- Unified data model over the synced records with entity resolution (people, time).
- Keyword search, semantic search (pgvector), and hybrid retrieval over synced records.
- Evidence engine: retrieved records become citable evidence items.
- Deterministic investigation workflow that plans, gathers evidence, builds a timeline,
  forms hypotheses, and produces a structured investigation report.
- A small set of agent tools for scoped retrieval.
- Investigation UI: submit a problem, watch the investigation, read the cited report.
- Permission-aware retrieval so the LLM never receives data the requesting user is not
  authorized to see.
- Observability, cost controls, and a small evaluation dataset.

## 10. V1 Non-Goals

- Not a generic enterprise search engine or a Glean/Slack-search replacement.
- No fully autonomous open-ended agent; the investigation workflow is deterministic first.
- No write-back to Slack/Jira/GitHub (read-only integrations).
- No data sources beyond Slack, Jira, GitHub.
- No on-prem / self-hosted LLM.
- No graph database, no streaming/event-bus infrastructure, no microservices.
- No fine-tuning of models.
- No mobile app.
- No SSO/SCIM/enterprise identity beyond basic auth for V1 (revisit later).

## 11. Supported Data Sources

- Slack (V1, first integration)
- Jira (V1)
- GitHub (V1)

## 12. Future Data Sources

Candidates, not committed: PagerDuty / Opsgenie, Datadog / Grafana / Sentry, Confluence /
Notion, Google Drive, Linear, GitLab, Zendesk, email.

## 13. Core User Workflow

1. Admin connects the organization's Slack, Jira, and GitHub via OAuth.
2. Initial sync pulls historical records; incremental sync keeps them current.
3. A user submits a problem statement (optionally with a time window / systems / people).
4. The investigation runs and streams progress.
5. The user reads the structured report and follows source links to verify each claim.

## 14. Investigation Workflow

```text
Question
  -> Investigation plan (what to look for, where, when)
  -> Tool calls / retrieval (permission-aware)
  -> Evidence collection (citable items)
  -> Cross-reference (link records across systems)
  -> Timeline construction
  -> Hypothesis generation (candidate root causes)
  -> Reasoning (weigh supporting vs contradictory evidence, assign confidence)
  -> Structured answer with citations and unresolved questions
```

## 15. Expected Output

A structured investigation report containing: Executive Summary, Root Cause, Confidence,
Timeline, Evidence, Related Issues, People Involved, Supporting Evidence, Contradictory
Evidence, Unresolved Questions, and Source Links. Every conclusion links to the underlying
records.

## 16. Product Principles

- Evidence first: no conclusion without a traceable source.
- Show the work: expose the timeline, the hypotheses, and the reasoning, not just an answer.
- Honest uncertainty: state confidence, surface contradictions and unresolved questions.
- Respect permissions: never reveal or reason over data the user cannot access.
- Narrow and deep: engineering investigation done well beats shallow generic search.

## 17. Technical Philosophy

- Incremental development: small, explicit, testable, reviewable, independently
  completable tasks.
- No premature complexity: simplest architecture that meets the requirement.
- No speculative implementation: build only what the current task needs.
- Preserve architectural consistency: inspect the repo and prior decisions before changing.
- Verify third-party APIs against official docs; never guess.

## 18. High-Level Architecture

Modular monolith. Next.js frontend talks to a FastAPI backend. FastAPI uses PostgreSQL
(with pgvector) as the system of record and vector store, Redis + a worker for background
sync and investigation jobs, and hosted LLM APIs. Integration connectors (Slack, Jira,
GitHub) run inside the backend. An investigation engine orchestrates retrieval, evidence,
timeline, and reasoning. See `docs/ARCHITECTURE.md`.

## 19. Major Components

- Web app (Next.js)
- API layer (FastAPI)
- Auth & tenancy module
- Connections & OAuth token store
- Sync engine + job queue (Redis + worker)
- Source connectors: Slack, Jira, GitHub
- Unified data model + entity resolution
- Search: keyword + semantic (pgvector) + hybrid + reranking
- Evidence engine (provenance)
- Investigation engine (plan / gather / cross-reference / timeline / reason)
- Agent tools
- Permission-aware retrieval layer
- Observability & cost controls

## 20. Data Flow

```text
OAuth connect -> tokens stored (encrypted) -> sync jobs enqueued
  -> connectors pull records -> raw + normalized rows in PostgreSQL
  -> chunk + embed -> embeddings in pgvector
User question -> investigation job -> permission-aware retrieval (keyword + vector)
  -> evidence items -> cross-reference + timeline -> hypotheses + reasoning
  -> structured report with citations -> stored + streamed to UI
```

## 21. AI/RAG Architecture

Query understanding -> parallel keyword and semantic search -> candidate set ->
metadata filtering (tenant, permissions, time, system) -> reranking -> evidence.
RAG uses per-source chunking with rich metadata, hybrid retrieval, and explicit context
construction. The investigation agent is given a bounded toolset and a maximum iteration
count. See `docs/AI_SYSTEM.md`.

## 22. Evidence/Provenance Philosophy

Every retrieved record that informs a conclusion is captured as an evidence item with a
stable reference to its source object (system, id, URL, author, timestamp, quoted span).
Investigation output cites evidence items by id. Evidence carries a level: Confirmed /
Likely / Possible / Unknown. Contradictory evidence is recorded, not discarded.

## 23. Security Philosophy

```text
User -> Identity -> Authorization -> Permission-aware retrieval
     -> Authorized evidence -> LLM
```

The LLM must never receive information the requesting user is not authorized to access.
OAuth tokens are encrypted at rest and never logged. All data access is tenant-scoped.
See `docs/SECURITY.md`.

## 24. Multi-Tenancy

One logical tenant = one `organization`. Every domain table carries `organization_id`
and all queries are filtered by it. Connections, synced records, embeddings, evidence,
and investigations are all tenant-scoped. Cross-tenant access is a security incident.
V1 uses shared-schema row-level scoping in application code (revisit RLS later — ADR).

## 25. V1 Implementation Roadmap

See `tasks/TODO.md` for the full checklist. Phases:

```text
0  Product & Architecture Foundation      (this initialization)
1  Project Foundation
2  Authentication & Multi-Tenancy
3  PostgreSQL Data Layer
4  Slack Connector
5  Background Job System
6  Jira Connector
7  Unified Data Model
8  Entity Resolution
9  Search Engine
10 Semantic Search / RAG
11 Hybrid Retrieval
12 Evidence Engine
13 GitHub Connector
14 Cross-System Relationships
15 Investigation Engine
16 Agent Tools
17 Investigation Planning
18 Timeline Engine
19 Reasoning & Root Cause
20 Citation / Source System
21 Permissions
22 Investigation UI
23 Hallucination & Quality Controls
24 Observability
25 Security
26 Evaluation Dataset
27 Cost Controls
28 V1 Production Deployment
29 V1 Beta
30 V1 Definition of Done
```

## 26. Current Phase

Phase 0 — Product & Architecture Foundation.

## 27. Current Task

No implementation task currently active. See `tasks/CURRENT_TASK.md`.

## 28. Completed Work

- Project documentation structure and memory system (this initialization).

No product functionality has been implemented yet.

## 29. Known Problems

None yet (no implementation).

## 30. Open Questions

- Exact hosted LLM provider/model for V1 (decide at Phase 10/15).
- Whether to adopt PostgreSQL row-level security in V1 or defer (see ADR-004 note).
- Jira: Cloud only for V1, or also Data Center? (Assume Cloud; verify at Phase 6.)
- GitHub: OAuth App vs GitHub App for V1 (see `docs/INTEGRATIONS.md`; decide at Phase 13).
- How permissions map from each source system to our retrieval filter (Phase 21).
- Reranking approach: hosted reranker API vs LLM-based vs lexical fusion only (Phase 11).

## 31. Architecture Decisions

Recorded in `docs/DECISIONS.md`:

- ADR-001 Initial architecture: modular monolith for V1.
- ADR-002 PostgreSQL as the primary database.
- ADR-003 pgvector for vector search (no separate vector DB in V1).
- ADR-004 Modular monolith over microservices for V1.
- ADR-005 Deterministic investigation workflow before autonomous agent behavior.
- ADR-006 Slack is the first integration.
- ADR-007 V1 focuses on engineering investigation, not generic enterprise search.

## 32. Future Vision

Broader source coverage, cross-source knowledge graph, proactive incident briefs,
decision/change history for any system, org-wide "why" queries, and eventually a more
autonomous investigator that can be trusted with open-ended questions.

## 33. Change Log

- 2026-08-31 — Project initialization. Created documentation structure, CLAUDE.md,
  PROJECT_MEMORY.md, all `docs/` files, `tasks/TODO.md`, `tasks/CURRENT_TASK.md`,
  README, `.gitignore`, `.env.example`. No product code.

---

## Current State

```text
Project initialization only.

No product functionality has been implemented yet.
```
