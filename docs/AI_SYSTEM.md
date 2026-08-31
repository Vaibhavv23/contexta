# AI_SYSTEM.md — Planned AI / Retrieval / Investigation Design

Status: planning only. Nothing here is implemented. Build incrementally from Phase 9
onward.

Related: [ARCHITECTURE.md](./ARCHITECTURE.md), [DATA_MODEL.md](./DATA_MODEL.md),
[SECURITY.md](./SECURITY.md), [PRODUCT.md](./PRODUCT.md).

---

## Retrieval Pipeline

```text
User Question
     ↓
Query Understanding      (intent, entities, time window, systems, people)
     ↓
Keyword Search   +   Semantic Search      (run in parallel)
     ↓
Candidate Results        (union of both result sets)
     ↓
Metadata Filtering       (tenant, permissions, time, system, container)
     ↓
Reranking                (order by relevance to the question)
     ↓
Evidence                 (top items become citable evidence)
```

**Query understanding** extracts: the core problem, named entities (services, tickets,
repos, people), an explicit or inferred time window, and which systems are likely
relevant. Output is a structured query object used by every downstream retriever.

**Permission filtering is not optional and not last-resort** — see
[SECURITY.md](./SECURITY.md). The candidate set is filtered to authorized records before
anything reaches the LLM.

---

## RAG

**Embeddings**
- One hosted embedding model (chosen at Phase 10). Model id recorded on each row.
- Re-embedding is a background job when the model or chunking changes.

**Chunking**
- Per-source strategy:
  - Slack: thread-aware; a thread (or a windowed slice of a busy channel) is a chunk unit.
  - Jira: issue summary+description as one unit; each comment as its own unit; history
    summarized.
  - GitHub: PR title+body as a unit; each review and comment as a unit; commit message as
    a unit.
- Target a bounded token size per chunk with small overlap where prose is continuous.

**Metadata** (stored with every embedding, used for filtering and citation)
- `organization_id`, `source`, `object_type`, `external_id`, `author_person_id`,
  `occurred_at`, `container_ref` (channel/project/repo), `url`.

**Vector search**
- pgvector similarity (cosine) with metadata pre-filtering by tenant + permissions +
  time + system.

**Hybrid search**
- Run keyword (PostgreSQL FTS) and vector search; fuse results (reciprocal rank fusion
  or weighted score). Keeps exact-match recall (ticket keys, error strings) that pure
  vector search misses.

**Reranking**
- Reorder the fused candidate list against the question. Approach options: hosted
  reranker API, LLM-based scoring, or fusion-only for V1. Decision at Phase 11.

**Context construction**
- Explicit assembly of a size-bounded context block from the top evidence items only.
- Each included item is labeled with its source, author, timestamp, and a stable id the
  model must use when citing.
- Never include raw un-filtered search output; never exceed the configured token budget.

---

## Investigation Agent

The agent operates a **bounded toolset** inside a **deterministic workflow** (ADR-005).
The workflow decides *what stage we are in*; the LLM decides *which tool calls to make
within a stage* and *how to phrase findings*.

**Planned tools**

```text
search_slack           keyword+semantic search over authorized Slack records
search_jira            keyword+semantic search over authorized Jira records
search_github          keyword+semantic search over authorized GitHub records
get_slack_thread       full thread by channel + root ts
get_jira_issue         issue with comments and history
get_github_pr          PR with reviews, comments, linked issues
get_commit             commit by repo + sha
find_related_entities  relationships for a given record (Phase 14)
search_by_person       records authored/involving a person in a window
search_by_time         records across systems within a time window
get_evidence           re-fetch an evidence item by id
```

All tools:
- accept the current `investigation_id` and run under the requesting user's identity;
- return only tenant-scoped, permission-authorized data;
- record an `investigation_events` entry for the call.

**Bounds**
- Maximum tool iterations per investigation (configurable; default small).
- Maximum evidence items.
- Maximum total tokens / cost per investigation (enforced; ties to Phase 27).
- Wall-clock timeout.

---

## Investigation Flow

```text
Question
 ↓
Investigation Plan       (stages, target systems, time window, key entities)
 ↓
Tool calls               (gather within the plan, bounded)
 ↓
Evidence                 (each useful result recorded with provenance + level)
 ↓
Cross-reference          (link records across systems via relationships)
 ↓
Timeline                 (order events; mark alert / diagnosis / change / resolution)
 ↓
Hypotheses               (candidate root causes, each tied to evidence)
 ↓
Reasoning                (weigh supporting vs contradictory evidence; assign confidence)
 ↓
Answer                   (structured report + prose, every claim cited)
```

---

## Evidence Levels

| Level | Meaning |
| --- | --- |
| Confirmed | Directly stated in a source record, or corroborated by ≥2 independent sources. |
| Likely | Strongly implied by evidence; minor gaps; no contradicting evidence. |
| Possible | Plausible and supported by some evidence, but incomplete or partly contradicted. |
| Unknown | The available data does not answer this; recorded as an unresolved question. |

The overall investigation `confidence` is derived from the level of the evidence behind
the stated root cause and the presence of contradictory evidence.

---

## Hallucination Prevention

- **Source grounding** — the model may only assert what is present in provided evidence
  items; the system prompt forbids outside knowledge for factual claims.
- **Citations** — every conclusion, timeline entry, and named person references one or
  more evidence ids. The report assembler rejects/flags uncited claims.
- **Evidence requirements** — no "Root Cause" section is emitted at Confirmed/Likely
  without linked supporting evidence.
- **Confidence** — always stated; low-evidence answers must say so.
- **Contradiction detection** — evidence that conflicts is surfaced in the
  "Contradictory Evidence" section, not dropped.
- **Unresolved questions** — gaps are listed explicitly.
- **Maximum agent iterations** — hard cap prevents runaway loops and speculative drift.
- **Evaluation dataset** (Phase 26) — seeded scenarios with known answers; each release
  is scored on correctness, citation validity, and permission compliance.

---

## Open AI Questions

- Hosted LLM provider/model for generation (Phase 15) and embeddings (Phase 10).
- Reranking approach (Phase 11).
- Whether query understanding is one LLM call or a cheaper classifier + rules.
- Prompt-caching strategy for the investigation system prompt and tool schemas.
