# PRODUCT.md — V1 Product Definition

Status: planning. Nothing in this document is implemented yet.

Related: [PROJECT_MEMORY.md](../PROJECT_MEMORY.md), [ARCHITECTURE.md](./ARCHITECTURE.md),
[AI_SYSTEM.md](./AI_SYSTEM.md), [SECURITY.md](./SECURITY.md).

---

## Product

The product investigates engineering and business problems by searching connected
company systems and synthesizing evidence into a coherent, cited explanation.

A user describes a problem in natural language. The system:

1. Interprets the problem and builds an investigation plan.
2. Retrieves relevant records from connected systems (Slack, Jira, GitHub), filtered to
   what the requesting user is authorized to see.
3. Turns those records into evidence items with full provenance.
4. Cross-references records across systems (a Slack thread that mentions a Jira key; a PR
   that closes an issue; a commit referenced in an incident channel).
5. Builds a timeline.
6. Generates candidate root-cause hypotheses.
7. Weighs supporting and contradictory evidence, assigns confidence, and produces a
   structured report where every conclusion links to its sources.

## Core V1 Question

The product is built to answer questions of this shape:

```text
Why did checkout fail yesterday?
What caused this production incident?
What changed before the incident?
Who identified the issue?
Which code change is likely responsible?
What happened between the first alert and resolution?
What evidence supports the root cause?
```

These share a pattern: the answer requires correlating records from multiple systems
around a time window and a topic, and it must be defensible with sources.

## V1 Supported Systems

```text
Slack
Jira
GitHub
```

Read-only. One workspace/org per connection. See [INTEGRATIONS.md](./INTEGRATIONS.md).

## V1 Investigation Output

The investigation report eventually contains these sections:

| Section | Content |
| --- | --- |
| Executive Summary | 2–4 sentences: what happened and the most likely cause. |
| Root Cause | The best-supported explanation, stated plainly. |
| Confidence | Confirmed / Likely / Possible / Unknown, with a one-line justification. |
| Timeline | Ordered events with timestamps and source links. |
| Evidence | The evidence items gathered, each with system, author, time, quoted span, link. |
| Related Issues | Jira issues / GitHub issues / PRs connected to the problem. |
| People Involved | Who reported, diagnosed, changed, reviewed, resolved. |
| Supporting Evidence | Evidence that supports the stated root cause. |
| Contradictory Evidence | Evidence that argues against it, or against discarded hypotheses. |
| Unresolved Questions | What the available data could not answer. |
| Source Links | Deep links to every cited record. |

Output is also available as structured data (for the UI and for evaluation), not only prose.

## What V1 Explicitly Does NOT Attempt To Solve

- Generic enterprise search / knowledge management across all company documents.
- Non-engineering investigations (HR, legal, finance, sales) as a first-class use case.
- Writing to or acting on Slack/Jira/GitHub (no comments, no ticket edits, no merges).
- Real-time alerting or incident automation / runbooks.
- Sources other than Slack, Jira, GitHub.
- Fully autonomous, open-ended agent reasoning without a deterministic workflow.
- Guaranteeing a definitive root cause — when evidence is insufficient, it says so.
- Historical data older than what the source APIs and the org's plan allow us to sync.
- Replacing human judgment in a retrospective; it produces a defensible draft.

## Success Criteria for V1

- For a seeded incident scenario, the investigation surfaces the correct responsible
  change and the key Slack/Jira/GitHub records, with citations, in a single run.
- No conclusion appears without at least one linked evidence item.
- No evidence item is drawn from data the requesting user cannot access.
- A user can click from any conclusion to the original record in the source system.
- Investigations complete within a bounded time and token/cost budget.
