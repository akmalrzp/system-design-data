---
type: pattern
title: "Technology requirements (neutral)"
tags: [proposal]
sources: [osdm-integration-summary,api-design-for-system-integration]
created: 2026-10-09
updated: 2026-10-21
---

# Technology requirements (neutral)

The guide does not pick products. Choose any technology that provides these capabilities. Examples are examples only.

| Capability | Requirement | Example (illustrative) |
|---|---|---|
| Atomic change plus outbox | Business change and outbox record in one local transaction | Relational transaction with an outbox table |
| Event channel | At-least-once delivery; ordering per entity key; retention long enough for catch-up and at least as long as a full bootstrap; consumer-side offset or equivalent | A message broker or log |
| Schema registry / contract store | Enforce compatibility; stored versions; FULL or FULL_TRANSITIVE preferred | A schema registry |
| Request/response contract | Machine-readable, reviewed before implementation | An OpenAPI document |
| Read API | Lookup by ID; cursor-paginated list with `updated_after`, sorted by (`updated_at`, `id`) | REST or RPC |
| Error format | Standard, machine-readable | `application/problem+json` |
| Idempotency | Idempotency key policy for retried writes (documented by the team; the IETF draft expired) | Header-based key |
| Concurrency | Optimistic concurrency on writes | ETag with `If-Match` (412 on mismatch) |
| Identity and authorisation | Per-consumer authorisation, field scoping | Team's standard; not researched |
| Observability | Trace and correlation IDs, lag metrics | Not researched |

Not covered by any source in the research: retries and backoff, dead-letter handling, circuit breakers, tracing, SLOs, OAuth/mTLS ([source-reliability](../ops/source-reliability.md)). Choose these deliberately and record them in an ADR ([data-dictionary-and-adr](../ops/data-dictionary-and-adr.md)).
