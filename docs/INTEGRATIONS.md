# INTEGRATIONS.md — Planned Integration Strategy

Status: planning. No integration is implemented. All source systems are **read-only**
in V1. One workspace/org per connection.

> **Rule 8 — No hallucinated APIs.** Every endpoint, scope, field name, rate limit,
> webhook payload, and auth detail below is provisional. Each item marked
> _"To be verified against current official API documentation during implementation."_
> must be checked against the live official docs before code is written.

Related: [DATA_MODEL.md](./DATA_MODEL.md), [SECURITY.md](./SECURITY.md),
[ARCHITECTURE.md](./ARCHITECTURE.md).

---

## Common Connector Design

- Each connector lives in `backend/app/connectors/<source>/`.
- Shared base handles: OAuth redirect + code exchange, token refresh, pagination,
  rate-limit handling (respect `Retry-After` / documented limits), retry with backoff,
  and mapping raw payloads to normalized rows + `source_objects`.
- Tokens stored encrypted in `connections` (see [SECURITY.md](./SECURITY.md)).
- Sync modes: **initial backfill** (bounded historical window) and **incremental**
  (cursor / updated-since / webhook-driven).
- Connectors never write to the source system.

---

## Slack

**OAuth**
- Slack OAuth v2 (`/oauth/v2/authorize`, `/api/oauth.v2.access`). Bot token, possibly
  user token for search.
- _To be verified against current official API documentation during implementation._

**Workspace**
- Fetch team info on connect; store in `slack_workspaces`.

**Users**
- Enumerate members; store id, names, email (if scope permits), bot/deleted flags.
- _Scopes for user listing and email visibility: to be verified against current official
  API documentation during implementation._

**Channels**
- List public channels the bot can see; private channels only if invited and scoped.
- Store name, type, privacy, archived state, topic/purpose.

**Messages**
- Fetch conversation history per channel; store text, author, ts, thread_ts, edits,
  reactions, permalink, and the raw payload.
- _History pagination, `oldest`/`latest` cursors, and rate-limit tiers: to be verified
  against current official API documentation during implementation._

**Threads**
- Fetch replies for messages with a thread; store thread root + reply metadata.

**Synchronization**
- Initial: backfill a bounded time window per channel.
- Incremental: poll history since last stored ts per channel, and/or consume the Events
  API. _Exact approach to be verified against current official API documentation during
  implementation._

**Incremental sync**
- Per-channel `cursor` / last-seen ts stored in `sync_jobs`.

**Rate limits**
- Slack applies per-method rate-limit tiers. Connector must honor them.
- _Specific tiers and limits: to be verified against current official API documentation
  during implementation._

**Webhooks / events**
- Slack Events API (URL verification, signed requests, event callbacks) as an option for
  near-real-time updates. _Event types and signature verification: to be verified against
  current official API documentation during implementation._

**Permissions**
- Coverage is bounded by granted scopes and the installing user's / bot's membership.
- Private channels and DMs are out of scope unless explicitly granted. Retrieval must
  respect per-user access (see [SECURITY.md](./SECURITY.md), Phase 21).

---

## Jira

**OAuth**
- Atlassian OAuth 2.0 (3LO) for Jira Cloud; authorization + token + refresh; cloudId
  resolution for API base URL.
- _To be verified against current official API documentation during implementation._

**Projects**
- List accessible projects; store key, name, id, type.

**Issues**
- Search/enumerate issues (JQL); store key, type, status, priority, summary,
  description, reporter, assignee, timestamps, labels, components, fix versions, raw.
- _JQL search endpoint, pagination, and field expansion: to be verified against current
  official API documentation during implementation._

**Comments**
- Fetch comments per issue; store author, body, timestamps.

**History**
- Fetch issue changelog; store field-level transitions with author and timestamp.
- _Changelog access (expand vs dedicated endpoint) and pagination: to be verified against
  current official API documentation during implementation._

**Synchronization**
- Initial: backfill issues per project (optionally bounded by updated date).
- Incremental: query issues `updated >= last_sync` via JQL; refetch comments/history for
  changed issues.

**Permissions**
- Respect Jira project/issue-level permissions and security schemes. A user in our app
  should only see evidence from issues they could see in Jira.
- _Permission model mapping: to be verified against current official API documentation
  during implementation_ (Phase 21).

---

## GitHub

**OAuth / App authentication**
- Two options: **OAuth App** (user token) or **GitHub App** (installation token, finer
  permissions, higher rate limits). Decision recorded via ADR at Phase 13.
- _Auth flow, token types, and installation model: to be verified against current
  official API documentation during implementation._

**Organizations**
- Resolve the connected org / owner; store in `github_organizations`.

**Repositories**
- List repos accessible to the token/installation; store name, privacy, default branch,
  archived state.

**Issues**
- Fetch issues per repo; store number, title, body, state, labels, author, assignees,
  timestamps, raw.

**Pull requests**
- Fetch PRs per repo; store number, title, body, state, merged flag/time, base/head,
  author, requested reviewers, linked issues, raw.

**Reviews**
- Fetch reviews per PR; store reviewer, state, body, submitted_at.

**Comments**
- Fetch issue comments, PR comments, and review (diff) comments; store parent ref,
  author, body, timestamps.

**Commits**
- Fetch commits per repo/branch (bounded window); store sha, author/committer, message,
  timestamps, parents, and a files/stats summary.
- _Commit listing pagination and diff/stat retrieval: to be verified against current
  official API documentation during implementation._

**Webhooks**
- Repo/org webhooks (issues, pull_request, pull_request_review, issue_comment, push) with
  signature verification, as an option for near-real-time updates.
- _Event payload shapes and signature scheme: to be verified against current official API
  documentation during implementation._

**Permissions**
- Respect repo visibility and the token/installation's repo access. Private repo data is
  only retrievable for users authorized for that repo (Phase 21).

**Rate limits**
- REST and GraphQL have separate quotas; secondary rate limits apply.
- _Exact limits and reset behavior: to be verified against current official API
  documentation during implementation._

---

## Cross-System Linking (Phase 14)

Signals used to build `relationships` rows:

- Jira issue keys mentioned in Slack messages, GitHub PR/issue titles/bodies, commit
  messages.
- GitHub "closes #N" / "fixes #N" links between PRs and issues.
- URLs to Slack/Jira/GitHub pasted in any system.
- Same person across systems (via entity resolution, Phase 8).
- Time-window co-occurrence around an incident.
