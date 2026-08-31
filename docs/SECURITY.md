# SECURITY.md — Planned Security Model

Status: planning. Nothing here is implemented. Security work is threaded through the
whole roadmap and consolidated in Phase 21 (Permissions) and Phase 25 (Security).

Related: [ARCHITECTURE.md](./ARCHITECTURE.md), [INTEGRATIONS.md](./INTEGRATIONS.md),
[AI_SYSTEM.md](./AI_SYSTEM.md), [DATA_MODEL.md](./DATA_MODEL.md).

---

## Core Principle

```text
User
 ↓
Identity            (who is making the request)
 ↓
Authorization       (what this user may see, per tenant and per source)
 ↓
Permission-aware retrieval   (retrieval filters to authorized records only)
 ↓
Authorized evidence
 ↓
LLM
```

**The LLM must never receive information the requesting user is not authorized to
access.** This constraint outranks answer quality. If authorized evidence is
insufficient, the investigation says so.

---

## Authentication

- Email/password or magic link (decide Phase 2), issuing a signed, HTTP-only, secure
  session cookie with a bounded lifetime.
- Passwords hashed with a memory-hard algorithm (argon2/bcrypt).
- No third-party SSO in V1; design leaves room for OIDC later.
- `current_user` dependency on every authenticated route.

## Authorization

- Roles per membership: owner / admin / member.
- Admin/owner: manage connections, members, and org settings.
- Member: run investigations, view investigations they are allowed to see.
- Every request resolves `(user, organization, role)` and checks it before acting.
- Source-level authorization (which Slack channels / Jira projects / GitHub repos a user
  may see) is computed for retrieval — see "Source Permissions".

## Multi-Tenancy

- Tenant = `organization`. `organization_id` on every domain row.
- All queries filtered by `organization_id` via a shared tenancy helper; services never
  issue unscoped queries.
- Cross-tenant data exposure is treated as a security incident.
- Candidate defense-in-depth: PostgreSQL row-level security (open question; ADR if adopted).

## OAuth Token Security

- Tokens (`access`, `refresh`) stored encrypted at rest using app-level envelope
  encryption; data key wrapped by a key from the secrets manager / KMS.
- Tokens decrypted only in memory, only when making a source API call.
- Tokens never returned by any API response, never rendered in the UI, never logged.
- Token refresh handled by the connector; failures mark the connection `needs_reauth`.
- On disconnect, tokens are deleted and (where the API supports it) revoked.

## Encryption

- In transit: TLS everywhere (client↔API, API↔worker if networked, API↔external APIs,
  API↔DB/Redis).
- At rest: managed DB/volume encryption; plus app-level encryption for OAuth tokens and
  any other designated secret fields.

## Secrets Management

- All secrets from environment variables / secrets manager. Never committed.
- `.env` is git-ignored; only `.env.example` (placeholders) is committed.
- No secret is ever written to logs, error messages, traces, or investigation events.
- CI/deploy inject secrets at runtime.

## Source Permissions

- For each source, compute the set of containers (Slack channels, Jira projects, GitHub
  repos) and objects the requesting user is authorized to see.
- V1 approach (to be refined at Phase 21): derive authorization from source membership /
  visibility where the API exposes it; conservative default is deny.
- Retrieval filters candidates by this set before reranking and before the LLM.
- Private/restricted content the app synced for one purpose is not exposed to users who
  lack access in the source system.

## Data Isolation

- Per-tenant logical isolation in a shared schema (V1).
- Embeddings, evidence, investigations, and relationships all carry `organization_id`
  and are filtered on every read.
- Vector search applies the tenant + permission filter as a pre-filter, not a
  post-filter.

## Audit Logs

- Append-only audit records for: login, connection connect/disconnect/reauth, sync start
  /finish, investigation start/finish, and evidence/data access within an investigation.
- Audit entries include actor, tenant, action, target, timestamp, and request id.
- No secrets or full message bodies in audit logs.

## LLM Data Boundaries

- Only authorized evidence items are placed in LLM context.
- Context is size-bounded and explicitly assembled (see [AI_SYSTEM.md](./AI_SYSTEM.md)).
- No raw OAuth tokens, secrets, or other tenants' data can enter a prompt by construction.
- LLM provider data-use terms reviewed before choosing a provider (no training on our
  data; retention understood). Recorded via ADR at Phase 15.
- Prompts and completions may be logged for debugging **only** with evidence bodies
  redacted/hashed unless the tenant opts in.

## Logging Restrictions

Never log: OAuth tokens, session cookies, passwords/hashes, encryption keys, full
message/issue/PR bodies (log ids + metadata), or PII beyond what is needed for
correlation. Structured logs use ids and reference keys.

## Data Retention

- Synced source data retained while the connection is active, subject to a configurable
  per-tenant retention window.
- Investigation reports and their evidence retained per tenant policy.
- Raw payloads retained for reprocessing but subject to the same retention window.

## Data Deletion

- Tenant offboarding deletes all tenant rows (synced data, embeddings, evidence,
  investigations, connections, audit beyond legal minimum) within a defined SLA.
- Per-connection deletion removes that source's synced data and embeddings.
- Deletion is verifiable (row counts to zero; background purge job).

## Integration Disconnect

- Disconnecting a source: stop syncs, delete/revoke tokens, and (per tenant choice)
  either retain or purge already-synced data for that source.
- Investigations that cited now-deleted evidence keep the report text but mark broken
  source links.

## Threat Model (initial)

| Threat | Mitigation |
| --- | --- |
| Cross-tenant data access | Mandatory `organization_id` scoping; tenancy helper; tests; consider RLS. |
| User sees source data they lack access to | Permission-aware retrieval pre-filter; conservative deny default. |
| OAuth token theft | Encrypted at rest; never logged/returned; least-scope; revoke on disconnect. |
| Secret leakage via logs | Logging allowlist; redaction; review. |
| Prompt injection via synced content | Treat source content as untrusted data, not instructions; the workflow — not the model — controls stages and tool access; output must be cited. |
| LLM provider retains/trains on data | Provider due diligence (ADR); redacted debug logging; opt-in only. |
| Excessive cost / runaway agent | Hard iteration, token, and wall-clock caps (Phase 27). |
| Broad OAuth scopes | Request minimum scopes; document each scope's purpose. |
| Insider access to prod data | Audit logs; least privilege; no secrets in code. |

## Not In Scope for V1

SSO/SCIM, customer-managed encryption keys, on-prem deployment, SOC2 audit
(design toward it, don't certify), field-level per-user redaction beyond
container-level permissions.
