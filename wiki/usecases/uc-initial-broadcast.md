---
type: usecase
title: "Use case: initial master broadcast to empty caches"
tags: [proposal]
sources: [osdm-integration-summary,api-design-for-system-integration]
created: 2026-10-09
updated: 2026-10-21
---

# Use case: initial master broadcast to empty caches

**Trigger**: a new owner goes live, or a consumer is added with an empty cache.

**Steps**
1. Order the entities by dependency (referenced before referencing; see [edge-case-circular-dependencies](../modules/edge-case-circular-dependencies.md)).
2. Owner populates its master with IDs and `version`.
3. Consumers record the event-stream position, then backfill through the list endpoint ([flow-bootstrap-and-catch-up](../patterns/flow-bootstrap-and-catch-up.md)), or the owner replays current state as events (one event per record, current version).
4. Consumers apply through the idempotent path ([rule-idempotent-versioned-apply](../patterns/rule-idempotent-versioned-apply.md)).
5. Reconcile ([flow-reconciliation](../patterns/flow-reconciliation.md)).
6. Only then enable validations that depend on the cache (for example assignments).

**Effect by module type**
- Owner ([data-owner-broadcaster](../modules/data-owner-broadcaster.md)): serves list and events; no schema change.
- Consumer type 1 ([consumer-attribute-user](../modules/consumer-attribute-user.md)): cache fills; placeholders resolve; no business impact.
- Consumer type 2 ([consumer-index-user](../modules/consumer-index-user.md)): validate that existing index records (D's requirements) match cached IDs; list unmatched ones for review.

**Effect by table type**: master gains rows; cache gains rows with `source_version`; transaction tables unchanged; projections are rebuilt ([table-projection](../tables/table-projection.md)).

**Edge cases**: records arriving before their parent (hierarchy: store, resolve later); large volume (page by cursor on `updated_at` plus `id`); events arriving mid-backfill (consumed from the recorded position; version guard discards older).
