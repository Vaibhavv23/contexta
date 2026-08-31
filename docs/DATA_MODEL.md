# DATA_MODEL.md — Planned Entities

Status: planning specification. **The database schema is not implemented.** This
document describes the intended entities, not final columns. Actual schema is created
incrementally via Alembic migrations starting at Phase 3.

Related: [ARCHITECTURE.md](./ARCHITECTURE.md), [INTEGRATIONS.md](./INTEGRATIONS.md),
[AI_SYSTEM.md](./AI_SYSTEM.md), [SECURITY.md](./SECURITY.md).

---

## Conventions (planned)

- Primary keys: UUID.
- Every domain table has `organization_id` (FK → `organizations`) and is always queried
  with it in the filter.
- Timestamps: `created_at`, `updated_at` (UTC). Source-native timestamps kept separately.
- Raw source payloads stored as JSONB (`raw`) alongside normalized columns.
- Soft delete where source deletion must be represented (`deleted_at`).
- External identifiers stored as `external_id` (string) + `source` enum; unique per
  `(organization_id, source, external_id, object_type)`.

---

## Core

| Table | Purpose | Key fields (planned) |
| --- | --- | --- |
| `organizations` | Tenant root. | `name`, `slug`, `created_at` |
| `users` | A person who can log in. | `email`, `password_hash` / auth ref, `name` |
| `memberships` | User ↔ organization, with role. | `user_id`, `organization_id`, `role` (owner/admin/member) |
| `connections` | An OAuth connection to a source system. | `organization_id`, `source` (slack/jira/github), `status`, `external_account_id`, `scopes`, `encrypted_token`, `encrypted_refresh_token`, `token_expires_at`, `connected_by_user_id` |
| `sync_jobs` | A unit of sync work and its status. | `organization_id`, `connection_id`, `job_type`, `resource`, `status`, `cursor`, `stats`, `started_at`, `finished_at`, `error` |

---

## Slack

| Table | Purpose |
| --- | --- |
| `slack_workspaces` | Connected Slack workspace (team) metadata. |
| `slack_users` | Slack members (id, name, real name, email if available, bot flag, deleted flag). |
| `slack_channels` | Channels (id, name, type, is_private, is_archived, topic, purpose). |
| `slack_messages` | Messages (channel, user, ts, text, thread_ts, edited, subtype, reactions, permalink, `raw`). |
| `slack_threads` | Thread roots and reply metadata (root ts, reply count, participants, last reply ts). |

Notes: private channel and DM coverage depends on granted scopes and installer
membership — to be verified against current Slack API docs at Phase 4.

---

## Jira

| Table | Purpose |
| --- | --- |
| `jira_projects` | Projects (key, name, id, project type). |
| `jira_users` | Jira users / account ids (accountId, displayName, email if visible). |
| `jira_issues` | Issues (key, id, project, type, status, priority, summary, description, reporter, assignee, created, updated, resolutiondate, labels, components, fix versions, `raw`). |
| `jira_comments` | Issue comments (issue, author, body, created, updated). |
| `jira_issue_history` | Changelog entries (issue, author, field, from, to, timestamp). |

Notes: Jira Cloud assumed for V1. Field availability and changelog access to be verified
against current Atlassian docs at Phase 6.

---

## GitHub

| Table | Purpose |
| --- | --- |
| `github_organizations` | Connected GitHub org / owner. |
| `github_users` | GitHub accounts (login, id, name, email if public). |
| `github_repositories` | Repos (name, full_name, id, private flag, default branch, archived). |
| `github_issues` | Issues (number, title, body, state, labels, author, assignees, created, closed, `raw`). |
| `github_pull_requests` | PRs (number, title, body, state, merged flag, merged_at, base/head, author, requested reviewers, linked issues, `raw`). |
| `github_reviews` | PR reviews (pr, reviewer, state, body, submitted_at). |
| `github_comments` | Issue comments, PR comments, and review comments (parent ref, author, body, created). |
| `github_commits` | Commits (sha, repo, author, committer, message, authored_at, committed_at, parents, files/stats summary). |

Notes: OAuth App vs GitHub App and the resulting permission model to be decided and
verified at Phase 13.

---

## AI / Search

| Table | Purpose | Key fields (planned) |
| --- | --- | --- |
| `source_objects` (a.k.a. `documents`) | Unified handle for any synced record that can be retrieved or cited. | `organization_id`, `source`, `object_type`, `external_id`, `title`, `text`, `author_person_id`, `occurred_at`, `url`, `container_ref` (channel/project/repo), `raw_ref` |
| `embeddings` | Vector embeddings of chunks of `source_objects`. | `organization_id`, `source_object_id`, `chunk_index`, `chunk_text`, `embedding` (pgvector), `metadata` (JSONB), `model`, `token_count` |
| `evidence` | A citable piece of evidence used in an investigation. | `organization_id`, `investigation_id`, `source_object_id`, `quoted_span`, `author_person_id`, `occurred_at`, `retrieval_reason`, `level` (confirmed/likely/possible/unknown) |
| `investigations` | One investigation run. | `organization_id`, `requested_by_user_id`, `question`, `params` (time window, systems, people), `status`, `plan`, `report` (structured JSONB), `summary`, `confidence`, `cost` (JSONB), `started_at`, `finished_at` |
| `investigation_events` | Step-by-step trace of an investigation. | `investigation_id`, `seq`, `type` (plan/tool_call/evidence/timeline/hypothesis/reasoning/report), `payload` (JSONB), `created_at` |
| `relationships` | A typed link between two records / entities across systems. | `organization_id`, `from_type`, `from_id`, `to_type`, `to_id`, `relation` (mentions / closes / references / fixes / duplicates / same_person / same_incident), `confidence`, `evidence_ref` |

`people` (entity-resolution table): a canonical person across sources, with links to
`slack_users`, `jira_users`, `github_users`. Created at Phase 8. Referenced above as
`author_person_id` / `*_person_id`.

---

## Relationship Notes

- `source_objects` is the join point between raw per-source tables and the search /
  evidence / investigation layers. Per-source tables remain the detailed record;
  `source_objects` is the retrievable/citable projection.
- `relationships` is a flat table, not a graph database (ADR: no Neo4j in V1).
- `evidence` rows are scoped to a single investigation and immutable once written.

---

## Not In This Document

Exact column types, indexes, constraints, and migration order. Those are defined in the
migration that introduces each table, task by task, per [DEVELOPMENT.md](./DEVELOPMENT.md).
